# Project State — CLI-like-a-Boss

> **Hva er denne filen?** Et permanent anker for kontekst. Hver gang vi starter en ny session (uansett CLI, uansett agent), leses denne filen først så vi vet hvor vi er, hva vi har bestemt, og hva som er neste steg. Oppdateres på slutten av hver økt.

---

## Tagline (én setning)

En multi-agent orkestrator der AI coding-CLI-er fra forskjellige leverandører kommuniserer med hverandre som agenter i et koordinert nettverk, med vedvarende hukommelse som gjør appen smartere over tid.

---

## Kjerneverdi (hva som faktisk skiller oss)

1. **Inter-agent-kommunikasjon** er produktet — ikke en bonus. Agenter fra ulike CLI-er sender meldinger til hverandre over en CLI-agnostisk protokoll, koordinerer arbeid, og bygger på hverandres output.
2. **Hukommelse** som blir bedre med bruk: episodisk, semantisk, prosedural, arbeidshukommelse. Reduserer både token-bruk og feilrater over tid.
3. **Brukerdefinert struktur + AI-anbefalinger underveis**. Du bygger arbeidsflyten, appen foreslår forbedringer basert på observerte utfall i din kodebase.

---

## Støttede CLI-er

**Fri / BYOK uten innlogging:**
- Vibe (Mistral) — gratis tier med API-nøkkel
- Aider — BYOK
- Continuum — gratis ubegrenset
- OpenCode (gratis) — gratis modeller / BYOK

**Med innlogging eller abonnement (OAuth-lignende flyt via UI-et):**
- Claude Code
- Codex CLI
- Cursor CLI
- Antigravity CLI (gratis 1000/dag)
- OpenCode Go (abonnement)

Brukeren kan veksle mellom fri og abonnement-modus per CLI der det er meningsfullt.

---

## Tech stack

- **Backend:** FastAPI (Python 3.10+)
- **Frontend:** Tauri + React + TypeScript + Tailwind
- **Database:** SQLite (lokal lagring)
- **Message bus:** Redis (pub/sub mellom agenter)
- **Vector store:** TBD — sqlite-vss eller Chroma for semantisk minne
- **Pakke:** pip (Python), npm (frontend)

---

## Arkitektur — kort versjon

```
┌─────────────────────────────────────────────┐
│              Tauri + React UI               │
│  (Workflow editor, dashboard, settings)     │
└──────────────────┬──────────────────────────┘
                   │ HTTP/WebSocket
┌──────────────────▼──────────────────────────┐
│           FastAPI Orchestrator              │
│  ┌─────────────┐  ┌───────────────────┐    │
│  │ Recommender │  │  Cost/Quality     │    │
│  │   Engine    │  │     Router        │    │
│  └─────────────┘  └───────────────────┘    │
│  ┌──────────────────────────────────────┐  │
│  │       Memory Service (4 layers)      │  │
│  │  episodic | semantic | procedural |  │  │
│  │            working                   │  │
│  └──────────────────────────────────────┘  │
└──────────────────┬──────────────────────────┘
                   │ Agent Protocol (over Redis pub/sub)
       ┌───────────┼───────────┬────────────┐
       ▼           ▼           ▼            ▼
   ┌──────┐   ┌────────┐  ┌──────────┐  ┌──────────┐
   │ Vibe │   │ Claude │  │ OpenCode │  │   ...    │
   │adapt │   │  Code  │  │  adapt   │  │  adapt   │
   │      │   │ adapt  │  │          │  │          │
   └───┬──┘   └───┬────┘  └────┬─────┘  └────┬─────┘
       │          │             │              │
       ▼          ▼             ▼              ▼
   [Vibe CLI] [Claude CLI] [OpenCode CLI]  [...CLI]
```

Detaljer: se `ARCHITECTURE.md`.

---

## Hukommelse-system (4 lag)

| Lag | Hva | Lagring |
|-----|-----|---------|
| **Episodisk** | Konkrete sesjoner: oppgave, agent, utfall, tid | SQLite tabell |
| **Semantisk** | Kodebase-kunnskap, konvensjoner, arkitekturvalg | Vector store + SQLite |
| **Prosedural** | "Playbooks": hvilke agent-kombinasjoner virker for hva | SQLite tabell, oppdatert av Recommender |
| **Arbeids** | Aktiv kontekst i pågående oppgave | Redis (kortvarig) |

Memory styres som en tjeneste — alle agenter skriver til og leser fra samme bank.

---

## Token-besparelse — strategier

1. **Semantisk cache** — like forespørsler returneres uten LLM-kall (cosine sim > 0.92).
2. **Rullende summarisering** av samtalehistorikk.
3. **RAG-injeksjon**: kun de 5–10 mest relevante minne-chunks per forespørsel.
4. **Cost/quality-router**: enkle oppgaver til billige/gratis CLI-er, komplekse til premium.
5. **Caveman-compression** (Julius Brussee) som valgfri profil — sterk på agent-til-agent-handoffs og memory file-injeksjoner, av som default for kode-generering. Krediteres i About.

---

## Acknowledgments (legges i UI About-seksjon)

- **Caveman** av Julius Brussee — token-komprimering for inter-agent-kommunikasjon. https://github.com/juliusbrussee/caveman
- **Mistral Vibe** — gratis tier CLI
- **OpenCode** — open source coding CLI
- **Aider** — BYOK coding CLI
- **Continuum** — gratis coding CLI

---

## Hvor vi er nå

- ✅ **Chunk 0:** GitHub-repo opprettet (`GaimsDevSoftware/CLI-like-a-Boss`), initial commit med README/LICENSE/.gitignore, mappestruktur.
- 🚧 **Chunk 1 (pågår):** Backend-skjelett — message bus, agent-protokoll, memory-service, adapter-base. Vibe-adapter som første konkrete implementasjon. Health-endepunkt for å verifisere boot.
- ⏳ **Chunk 2:** Flere adaptere (Claude Code, OpenCode) + recommender engine first pass + auth-modul.
- ⏳ **Chunk 3:** Tauri-frontend — workflow editor, dashboard.
- ⏳ **Chunk 4:** Caveman-integrasjon som compression profile, observability, optimalisering.

---

## Beslutninger som er låst

- Backend Python (ikke Rust/Go) — raskere prototyping, bedre LLM-økosystem.
- Frontend Tauri (ikke Electron) — lavere ressursbruk på Linux.
- Redis som message bus (ikke RabbitMQ/NATS) — i requirements allerede, enkel pub/sub, god til vår skala.
- SQLite (ikke Postgres) — lokal-først app, ingen server-avhengighet.
- Caveman som *valgfri profil*, ikke påtvunget — kode-generering forblir uncomprimert som default.

---

## Spørsmål som ennå ikke er låst

- Vector store-valg: `sqlite-vss` (en database) vs. Chroma (separat prosess). Tas i Chunk 2.
- Lisens på selve appen utover MIT (per LICENSE-fil). Greit som det er.
- Cross-CLI tool sharing via MCP — design utestående. Tas i Chunk 2 eller 3.

---

## Hvordan oppdatere denne filen

Når du avslutter en økt (eller før du går til ny CLI), endre seksjonene "Hvor vi er nå" og "Spørsmål som ennå ikke er låst". Commit `docs/STATE.md` som siste handling i økten. Da har neste session perfekt kontekst, uavhengig av hvilken agent som tar over.

