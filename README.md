# CLI-like-a-Boss

**A Multi-Agent Orchestration Network for AI Coding CLIs**

Connect and coordinate multiple AI coding CLI tools (Vibe, Claude Code, Antigravity, OpenCode, Aider, etc.) as autonomous agents in a distributed development network.

---

## 🚀 Quick Start

### Prerequisites
- Linux (tested on Mint Cinnamon, compatible with all major distros)
- Python 3.10+
- Node.js 18+ (for Tauri frontend)
- Rust (for Tauri)

### Installation

```bash
# Clone the repository
git clone https://github.com/your-username/CLI-like-a-Boss.git
cd CLI-like-a-Boss

# Setup backend (Python)
cd backend
pip install -r requirements.txt

# Setup frontend (Tauri)
cd ../frontend
npm install
```

---

## 📁 Project Structure

```
CLI-like-a-Boss/
├── backend/           # FastAPI server (Python)
│   ├── main.py        # FastAPI app entry point
│   ├── config.py      # Configuration management
│   ├── models.py      # Data models
│   ├── routes/        # API endpoints
│   │   ├── auth.py
│   │   ├── usage.py
│   │   └── tasks.py
│   └── services/      # Business logic
│       ├── cli_manager.py
│       ├── router.py
│       └── tracker.py
├── frontend/          # Tauri + React UI
│   ├── src/           # React source
│   ├── public/        # Static assets
│   └── tauri.conf.json
├── adapters/          # CLI adapter implementations
│   ├── base.py        # Base adapter interface
│   ├── vibe.py
│   ├── claude.py
│   ├── antigravity.py
│   ├── opcode.py
│   └── aider.py
├── scripts/           # Utility scripts
│   └── setup.sh
├── docs/              # Documentation
└── README.md
```

---

## ✨ Features

- **Multi-Agent Orchestration**: Coordinate multiple AI CLI tools as specialized agents
- **Role-Based Assignment**: Assign roles (Architect, Coder, Debugger, Documenter) to each CLI
- **Auth Management**: Secure storage and management of API keys
- **Usage Tracking**: Monitor free tier limits and distribute tasks optimally
- **Smart Routing**: Automatically route tasks to the best available CLI
- **Cross-Platform**: Built for Linux, compatible with all major distributions

---

## 🔑 Supported CLIs

| CLI | Status | Free Tier | Auth Required |
|-----|--------|-----------|----------------|
| Vibe | ✅ Supported | Yes (local/unlimited) | Optional (cloud) |
| Claude Code | ✅ Supported | No | Yes |
| Antigravity CLI | ✅ Supported | Yes (1000/day) | Yes |
| OpenCode | ✅ Supported | Yes (unlimited) | Optional |
| Aider | ✅ Supported | Yes (unlimited) | No (BYOK) |
| Codex CLI | ✅ Supported | No | Yes |
| Cursor CLI | ✅ Supported | No | Yes |
| Continuum | ✅ Supported | Yes (unlimited) | No |

---

## 🛠️ Tech Stack

- **Backend**: FastAPI (Python 3.10+)
- **Frontend**: Tauri + React + TypeScript
- **Database**: SQLite (local storage)
- **Styling**: Tailwind CSS
- **Package Management**: pip (Python), npm (Frontend)

---

## 📦 Dependencies

### Backend (Python)
```
fastapi>=0.104.0
uvicorn>=0.24.0
pydantic>=2.5.0
sqlalchemy>=2.0.23
python-dotenv>=1.0.0
redis>=5.0.0
cryptography>=41.0.0
```

### Frontend (Tauri)
```
@tauri-apps/api
react
react-dom
@tauri-apps/cli
```

---

## 🚀 Running the App

### Development Mode

```bash
# Terminal 1: Backend
cd backend
uvicorn main:app --reload --port 8000

# Terminal 2: Frontend
cd frontend
npm run tauri dev
```

### Production Build

```bash
# Build frontend
cd frontend
npm run tauri build

# Run backend
cd backend
uvicorn main:app --host 0.0.0.0 --port 8000
```

---

## 🔧 Configuration

Create a `.env` file in the backend directory:

```env
# Database
DATABASE_URL=sqlite:///./app.db

# Redis (for message bus)
REDIS_URL=redis://localhost:6379/0

# Encryption key (generate with: openssl rand -hex 32)
ENCRYPTION_KEY=your-32-byte-hex-key
```

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- Inspired by Maestri for Mac
- Built with Mistral Vibe
- Powered by the open-source community
