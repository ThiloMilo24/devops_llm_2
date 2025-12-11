# Rusty 2.0 - Repository Übersicht

## Zusammenfassung

**Rusty 2.0** ist ein autonomer Entwicklungs-Assistent (DevAgent), der mithilfe eines Large Language Models (LLM) selbstständig Programmier-Aufgaben in einem Git-Repository erledigen kann.

---

## Architektur-Überblick

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND                                  │
│  ┌─────────────────────┐    ┌─────────────────────────────┐     │
│  │   Streamlit UI      │    │   Swagger UI (auto)         │     │
│  │   (app.py)          │    │   /docs                     │     │
│  │   Port 8501         │    │   Port 8000                 │     │
│  └──────────┬──────────┘    └──────────────┬──────────────┘     │
└─────────────┼───────────────────────────────┼───────────────────┘
              │ HTTP                          │
              ▼                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        BACKEND                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    FastAPI (api.py)                      │    │
│  │                    /dev-agent/run                        │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                    │
│  ┌──────────────────────────▼──────────────────────────────┐    │
│  │                 DevAgent (dev_agent.py)                  │    │
│  │         Orchestriert LLM-Aufrufe und Tool-Ausführung     │    │
│  └─────┬─────────────────────────────────────────────┬─────┘    │
│        │                                             │          │
│        ▼                                             ▼          │
│  ┌───────────────┐                           ┌──────────────┐   │
│  │  LLM Client   │                           │ MCP Client   │   │
│  │ (OpenAI/Gemini)│                          │ (Git Tools)  │   │
│  └───────────────┘                           └──────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ordnerstruktur

```
devops_llm_2/
├── rusty_2/                    # Hauptpaket
│   ├── __init__.py
│   ├── backend/                # Backend-Logik
│   │   ├── __init__.py
│   │   ├── api.py              # FastAPI Server
│   │   ├── dev_agent.py        # Kern-Agent-Logik
│   │   ├── local_tools.py      # Datei-Tools
│   │   └── eval/               # Evaluations-Framework
│   │       ├── __init__.py
│   │       ├── metrics.py      # Evaluations-Metriken
│   │       ├── run_eval.py     # Evaluations-Runner
│   │       └── tasks.yaml.example
│   ├── common/                 # Gemeinsame Module
│   │   ├── __init__.py
│   │   ├── conversation.py     # Chat-Verlauf Management
│   │   ├── llm_client.py       # LLM API Abstraktion
│   │   ├── mcp_client.py       # MCP Server Client
│   │   ├── messages.py         # Message-Formate
│   │   ├── settings.py         # Konfiguration
│   │   └── unified_diff.py     # Diff-Patch-Logik
│   └── frontend/               # Frontend
│       ├── __init__.py
│       ├── app.py              # Streamlit Hauptapp
│       └── streamlit_display.py # Chat-Darstellung
├── scripts/                    # Test-Skripte
│   ├── test_agent.py
│   ├── test_llm.py
│   ├── test_local_tools.py
│   └── test_mcp_git.py
├── CICD/Docker/                # Docker (leer)
├── environment.yml             # Conda Dependencies
├── .env                        # API Keys (nicht in Git)
├── .gitignore
└── README.md
```

---

## Detaillierte File-Beschreibungen

### Backend (`rusty_2/backend/`)

#### `api.py` - FastAPI Server
**Zweck:** REST-API Schnittstelle für den DevAgent

**Endpoints:**
| Endpoint | Methode | Beschreibung |
|----------|---------|--------------|
| `/` | GET | API-Info |
| `/health` | GET | Health-Check |
| `/dev-agent/run` | POST | Führt eine Agent-Aufgabe aus |

**Verwendete Pakete:**
- `fastapi` - Web-Framework
- `pydantic` - Datenvalidierung

---

#### `dev_agent.py` - Kern-Agent-Logik
**Zweck:** Orchestriert die autonome Aufgabenbearbeitung

**Hauptklassen:**
| Klasse | Beschreibung |
|--------|--------------|
| `DevAgentConfig` | Konfiguration (max_steps, backend, etc.) |
| `DevAgentResult` | Ergebnis eines Agent-Laufs |
| `DevAgent` | Hauptklasse - führt Schritte aus |

**Wichtige Funktionen:**
| Funktion | Beschreibung |
|----------|--------------|
| `run_step()` | Führt einen Agent-Schritt aus (LLM fragen → Tool ausführen) |
| `run_task()` | Führt komplette Aufgabe aus (Schleife bis "Task completed") |
| `create_initial_conversation()` | Erstellt System-Prompt + User-Nachricht |

**Ablauf:**
1. User gibt Aufgabe ein
2. System-Prompt wird erstellt (definiert Agent-Rolle)
3. Schleife: LLM fragen → Tool-Calls ausführen → Ergebnis zurück ans LLM
4. Stopp wenn LLM "Task completed" sagt oder max_steps erreicht

---

#### `local_tools.py` - Lokale Datei-Tools
**Zweck:** Ermöglicht dem Agent, Dateien zu lesen und zu ändern

**Tools:**
| Tool | Beschreibung |
|------|--------------|
| `read_file` | Liest eine UTF-8 Datei |
| `apply_unified_diff` | Wendet einen Diff-Patch auf eine Datei an |

**Sicherheit:** Pfade werden validiert, um das Repository nicht zu verlassen.

---

### Common (`rusty_2/common/`)

#### `llm_client.py` - LLM API Abstraktion
**Zweck:** Einheitliche Schnittstelle für verschiedene LLM-Backends

**Unterstützte Backends:**
| Backend | API-Endpoint | Default-Modell |
|---------|--------------|----------------|
| OpenAI | api.openai.com | gpt-4o-mini |
| Gemini | generativelanguage.googleapis.com | gemini-pro |

**Features:**
- Rate-Limiting (requests per minute)
- Automatische Message-Normalisierung
- Tool-Calling Support

**Hauptklassen:**
| Klasse | Beschreibung |
|--------|--------------|
| `ModelConfig` | API-Konfiguration (Key, URL, Modell) |
| `ModelClient` | Führt API-Calls aus |

---

#### `mcp_client.py` - MCP Server Client
**Zweck:** Verbindet zum Model Context Protocol (MCP) Server für Git-Operationen

**Was ist MCP?**
Ein Protokoll von Anthropic, um LLMs mit externen Tools zu verbinden. Hier wird `mcp-server-git` verwendet.

**Verfügbare Git-Tools (via MCP):**
- `git_status` - Repository-Status
- `git_log` - Commit-Historie
- `git_diff` - Änderungen anzeigen
- `git_branch` - Branches verwalten
- etc.

**Verbindung:** Via `stdio://` URL (startet lokalen Prozess)

---

#### `conversation.py` - Chat-Verlauf Management
**Zweck:** Verwaltet die Nachrichten-Historie

**Features:**
| Feature | Beschreibung |
|---------|--------------|
| `append()` | Nachricht hinzufügen |
| `save()` / `load()` | Als JSON speichern/laden |
| `register_observer()` | Observer-Pattern für Live-Updates |

**Message-Rollen:**
- `system` - System-Prompt (Agent-Anweisungen)
- `user` - Benutzer-Nachrichten
- `assistant` - LLM-Antworten
- `tool` - Tool-Ergebnisse

---

#### `unified_diff.py` - Diff-Patch-Logik
**Zweck:** Parst und wendet Unified-Diff Patches an

**Was ist ein Unified Diff?**
```diff
--- original.py
+++ modified.py
@@ -1,3 +1,4 @@
 def hello():
-    print("Hello")
+    print("Hello World")
+    return True
```

**Klassen:**
| Klasse | Beschreibung |
|--------|--------------|
| `UnifiedDiff` | Ganzer Patch |
| `UnifiedDiffHunk` | Einzelner Änderungsblock |

---

#### `settings.py` - Konfiguration
**Zweck:** Lädt Umgebungsvariablen aus `.env`

**Umgebungsvariablen:**
| Variable | Beschreibung | Default |
|----------|--------------|---------|
| `OPENAI_API_KEY` | OpenAI API Key | - |
| `GOOGLE_API_KEY` | Gemini API Key | - |
| `LLM_BACKEND_NAME` | Backend-Auswahl | "openai" |
| `LLM_MODEL_NAME` | Modell-Name | "gpt-4o-mini" |
| `LLM_REQUESTS_PER_MINUTE` | Rate-Limit | 60 |

---

### Frontend (`rusty_2/frontend/`)

#### `app.py` - Streamlit Hauptapp
**Zweck:** Benutzeroberfläche für den DevAgent

**UI-Elemente:**
- Sidebar: Konfiguration (Repo-Pfad, MCP-URL, Max-Steps)
- Hauptbereich: Task-Eingabe + Run-Button
- Ergebnis: Erfolgs-Anzeige + Chat-Verlauf

**Flow:**
1. User füllt Formular aus
2. Klickt "Run Dev Agent"
3. HTTP-Request an FastAPI Backend
4. Ergebnis wird als Chat dargestellt

---

#### `streamlit_display.py` - Chat-Darstellung
**Zweck:** Rendert Chat-Nachrichten mit Styling

**Styling pro Rolle:**
| Rolle | Darstellung |
|-------|-------------|
| System | Grau, Info-Icon |
| User | Blau, rechtsbündig |
| Assistant | Grün, linksbündig |
| Tool | Monospace Code-Block |

---

### Evaluation (`rusty_2/backend/eval/`)

#### `run_eval.py` - Evaluations-Runner
**Zweck:** Automatisierte Evaluation von Agent-Tasks

**Workflow:**
1. Tasks aus `tasks.yaml` laden
2. Für jeden Task: Agent ausführen
3. Checks durchführen:
   - `check_compile()` - Kompiliert der Code?
   - `check_tests()` - Bestehen Unit-Tests?
   - `check_static()` - Linter-Checks (ruff/flake8)
4. Ergebnisse als CSV/JSON speichern

**Aufruf:**
```bash
python -m rusty_2.backend.eval.run_eval --tasks tasks.yaml --output-dir ./results
```

---

### Test-Skripte (`scripts/`)

| Script | Beschreibung |
|--------|--------------|
| `test_llm.py` | Testet LLM-Verbindung (OpenAI/Gemini) |
| `test_agent.py` | Testet kompletten Agent-Flow |
| `test_local_tools.py` | Testet read_file/apply_diff |
| `test_mcp_git.py` | Testet MCP Git-Server |

---

## Verwendete Pakete

| Paket | Version | Verwendung |
|-------|---------|------------|
| `python` | 3.11 | Runtime |
| `fastapi` | - | REST-API Server |
| `uvicorn` | - | ASGI Server für FastAPI |
| `streamlit` | - | Frontend UI |
| `openai` | - | LLM API Client |
| `mcp` | - | Model Context Protocol |
| `mcp-server-git` | - | Git-Tools via MCP |
| `requests` | - | HTTP Client |
| `python-dotenv` | - | .env Laden |
| `pyyaml` | - | YAML Parsing (Eval) |

---

## Datenfluss

```
┌──────────┐     Task       ┌──────────┐     HTTP      ┌──────────┐
│   User   │ ─────────────▶ │ Streamlit│ ────────────▶ │ FastAPI  │
└──────────┘                └──────────┘               └────┬─────┘
                                                            │
                            ┌───────────────────────────────┘
                            ▼
                      ┌───────────┐
                      │ DevAgent  │◀──────────────────────┐
                      └─────┬─────┘                       │
                            │                             │
           ┌────────────────┼────────────────┐            │
           ▼                ▼                ▼            │
     ┌──────────┐    ┌──────────┐    ┌──────────┐        │
     │ LLM API  │    │MCP Server│    │Local Tools│        │
     │(OpenAI/  │    │  (Git)   │    │(read/diff)│        │
     │ Gemini)  │    └────┬─────┘    └─────┬────┘        │
     └────┬─────┘         │                │             │
          │               │                │             │
          │    Tool Results                │             │
          └───────────────┴────────────────┴─────────────┘
```

---

## Quick Start

```bash
# 1. Environment erstellen
conda env create -f environment.yml
conda activate rusty_2

# 2. API Key setzen
echo "GOOGLE_API_KEY=dein-key" > .env
echo "LLM_BACKEND_NAME=gemini" >> .env

# 3. Backend starten (Terminal 1)
uvicorn rusty_2.backend.api:app --reload

# 4. Frontend starten (Terminal 2)
streamlit run rusty_2/frontend/app.py

# 5. Browser öffnen: http://localhost:8501
```

---

## Glossar

| Begriff | Erklärung |
|---------|-----------|
| **LLM** | Large Language Model (z.B. GPT-4, Gemini) |
| **MCP** | Model Context Protocol - Protokoll für Tool-Anbindung |
| **Agent** | Autonomes System das Aufgaben selbstständig bearbeitet |
| **Tool-Calling** | LLM ruft externe Funktionen auf |
| **Unified Diff** | Standard-Format für Code-Änderungen |
| **FastAPI** | Modernes Python Web-Framework |
| **Streamlit** | Framework für Data-Science UIs |
| **ASGI** | Async Server Gateway Interface |

---

*Erstellt für DevOps & LLMs - HSLU Master*
