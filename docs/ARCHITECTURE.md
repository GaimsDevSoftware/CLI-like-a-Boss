# Architecture — CLI-like-a-Boss

> Designdokumentet. Beskriver hvordan agenter snakker sammen, hvordan hukommelse er strukturert, og hvilke kontrakter som må holdes.

---

## 1. Agent-protokoll (A2A — Agent-to-Agent)

Alle agenter, uavhengig av hvilken CLI de driftes av, snakker over samme meldingsformat. Adapterne oversetter mellom protokollen og det enkelte CLI sitt native format.

### Meldingsstruktur

```json
{
  "id": "msg_01HXYZ...",
  "session_id": "sess_01HABC...",
  "from_agent": "agent_vibe_001",
  "to_agent": "agent_claude_002",
  "role": "request | response | broadcast | handoff",
  "intent": "review_code | implement | plan | research | summarize | ...",
  "payload": {
    "task": "...",
    "context_refs": ["mem:episodic:42", "file:src/foo.py:L10-L40"],
    "constraints": {"max_tokens": 4000, "deadline_ms": 30000}
  },
  "meta": {
    "created_at": "2026-05-27T18:00:00Z",
    "trace_id": "trace_01HABC...",
    "parent_msg_id": "msg_01HXYZ..."
  }
}
```

### Roller

- `request` — agent ber en annen agent gjøre noe
- `response` — svar på et request
- `broadcast` — informasjon til alle agenter i sesjonen
- `handoff` — full overlevering av eierskap (med oppsummert kontekst)

### Intents

Foreløpig liste (utvides etter behov):
`plan`, `implement`, `review_code`, `debug`, `research`, `summarize`, `test`, `document`, `refactor`.

Hver adapter erklærer hvilke intents den støtter best (brukes av Recommender).

### Context references

I stedet for å sende hele filer/historikk i `payload`, sender vi referanser til Memory-tjenesten. Mottaker-agent henter selv det den trenger. Det er nøkkelen til å holde meldinger små.

Eksempler:
- `mem:episodic:42` — hent episodisk minne #42
- `mem:semantic:topic="async patterns"` — semantisk søk
- `file:src/foo.py:L10-L40` — fil-utdrag
- `tool:mcp:github:issue#1234` — MCP-ressurs

---

## 2. Message bus (Redis)

- Hver sesjon har sin egen kanal: `session:{session_id}`
- Hver agent abonnerer på sin egen kø: `agent:{agent_id}`
- En `broadcast`-melding sendes på sesjonskanalen; `request`/`response` går til mottaker-agentens kø.
- Vi bruker Redis Streams (ikke pub/sub) for at meldinger skal overleve restart og kunne replays.

```
Redis streams:
  session:{sid}              # broadcast + observability
  agent:{aid}:inbox          # innkommende
  agent:{aid}:outbox         # utgående (for trace)
  memory:writes              # alle minne-skrivinger (for indeksering)
```

---

## 3. Adapter-kontrakt

Hver CLI har en adapter som implementerer dette grensesnittet:

```python
class CliAdapter(Protocol):
    name: str                           # "vibe", "claude_code", ...
    supported_intents: list[str]
    auth_mode: Literal["byok", "oauth", "none"]
    
    async def start(self, session_id: str, agent_id: str) -> None:
        """Start agent-prosessen, abonner på inbox."""
    
    async def handle_message(self, msg: AgentMessage) -> AgentMessage:
        """Bearbeide en innkommende melding, returner respons."""
    
    async def stop(self) -> None:
        """Rydd opp."""
    
    async def health(self) -> AdapterHealth:
        """Status: er CLI installert, auth ok, rate-limit?"""
```

Adapteren er ansvarlig for:
- Konvertere `AgentMessage.payload` til CLI-ets native input (stdin, args, API-kall)
- Konvertere CLI-ets output til en `AgentMessage` med korrekt struktur
- Hente kontekst fra Memory-tjenesten basert på `context_refs`
- Logge token-bruk til observability-laget

---

## 4. Memory-tjeneste

Singleton-service som alle agenter snakker med over et internt API (ikke gjennom message bus, for å unngå sirkulær avhengighet).

### Skjemaer (SQLite)

```sql
-- Episodisk: konkrete hendelser
CREATE TABLE episodes (
    id INTEGER PRIMARY KEY,
    session_id TEXT,
    agent_id TEXT,
    task TEXT,
    outcome TEXT,           -- "success" | "fail" | "partial"
    artifacts_json TEXT,    -- referanser til filer, diffs, etc.
    created_at TIMESTAMP,
    duration_ms INTEGER,
    tokens_used INTEGER
);

-- Semantisk: konsept-nivå kunnskap
CREATE TABLE semantic_facts (
    id INTEGER PRIMARY KEY,
    topic TEXT,
    statement TEXT,
    source TEXT,            -- "agent:claude" | "user" | "doc:..."
    confidence REAL,
    embedding BLOB,         -- vektorindeks
    created_at TIMESTAMP,
    last_referenced TIMESTAMP
);

-- Prosedural: "for type X bruk fremgangsmåte Y"
CREATE TABLE procedures (
    id INTEGER PRIMARY KEY,
    pattern TEXT,            -- "async race condition in tests"
    recommended_agent TEXT,
    recommended_role TEXT,
    success_rate REAL,
    trials INTEGER,
    last_updated TIMESTAMP
);
```

Arbeidshukommelse lever i Redis under `session:{sid}:context` og kastes når sesjonen lukkes (med mulighet for å promotere til episodisk minne hvis utfallet var verdt å huske).

### API

```python
class MemoryService:
    async def remember(self, kind: str, data: dict) -> str:        # returns mem-ref
    async def recall(self, ref: str) -> dict
    async def search_semantic(self, query: str, k: int = 5) -> list[Fact]
    async def update_procedure(self, pattern: str, outcome: str) -> None
    async def summarize_session(self, session_id: str) -> str
```

---

## 5. Recommender Engine

Observerer alle `episodes` og oppdaterer `procedures`-tabellen. Når en ny oppgave kommer inn, foreslår den (a) hvilken agent som er best for jobben, (b) hvilken arbeidsflyt (sekvens av agent-roller) som har høyest historisk suksessrate.

Stages:
1. **Klassifiser** oppgaven (LLM-basert, billig modell).
2. **Slå opp** matchende prosedyrer.
3. **Score** kandidater med (suksessrate × erfaringsantall × kostnad-faktor).
4. **Foreslå** topp 1–3 med begrunnelse. Brukeren bekrefter eller overstyrer.

Recommender lærer av: episode-utfall, brukerens tommel-opp/ned, agentens egen self-rapporterte sikkerhet.

---

## 6. Cost/Quality Router

Statisk tabell + dynamisk justering:

```python
ROUTING_PROFILES = {
    "free_first":     prioritize(cost_asc),
    "best_first":     prioritize(quality_desc),
    "balanced":       blend(cost, quality, latency, weights=[.4, .4, .2]),
    "subscription":   restrict_to(subscription_authed_clis),
}
```

Brukeren velger profil per workflow eller per oppgave. Routeren respekterer både profilen og Recommenderens forslag.

---

## 7. Caveman-integrasjon (compression profile)

Compression-laget sitter mellom adapter og message bus:

```
Agent A → [Adapter A] → [CompressionFilter] → MessageBus → [CompressionFilter] → [Adapter B] → Agent B
```

Konfigurasjon per kant i workflow-grafen:
- `none` (default for kode-generering)
- `lite` (fjerner hedging, behold tekniske termer)
- `medium` (caveman standard)
- `ultra` (caveman ultra — kun for handoffs mellom planlegger/oppsummerer)

For `memory:writes` kjører vi alltid `caveman-compress` slik at episodiske minner lagres komprimert. Henting dekomprimerer ikke (vi vil ha minner kompakt for fremtidig injeksjon).

---

## 8. Observability

Alt som går gjennom message bus replikeres til SQLite-tabellen `events` med `trace_id`. Dashboard viser per CLI / per agent-rolle / per workflow:

- Token-bruk (input + output + thinking hvis tilgjengelig)
- Suksessrate (basert på episode-outcome)
- P50/P95 latens
- Kostnad i kr/USD (med valuta-kurs hentet daglig)

Recommender bruker samme data — observability og læring deler underliggende tabeller.

---

## 9. Sikkerhet & secrets

API-nøkler og OAuth-tokens lagres i OS keychain (libsecret på Linux, Keychain på Mac), ikke i klartekst-konfig. Backend henter via `keyring`-biblioteket. Sensitive verdier maskeres i alle logger og frontend-displays.

---

## 10. Hva ikke er bestemt enda

- Vector store: `sqlite-vss` (innebygd) vs. Chroma (server). Avgjøres i Chunk 2 etter målinger.
- MCP-tool sharing mellom agenter — design utestående. Sannsynligvis en MCP-proxy-tjeneste som adapterne kobler til.
- Hvordan håndtere ulike CLI-er som har inkompatibel forventning til arbeidsmappe (Aider antar én ting, Claude Code en annen). Antagelig en sandboks-prosess per CLI.

