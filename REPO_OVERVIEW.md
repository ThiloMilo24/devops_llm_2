# Rusty 2.0 - Vollständige Repository Dokumentation

---

## 1. Was ist dieses Projekt?

**Rusty 2.0** ist ein **autonomer Entwicklungs-Assistent**. Das bedeutet:

- Du gibst ihm eine Aufgabe (z.B. "Fix den Bug in auth.py")
- Er analysiert den Code selbstständig
- Er macht Änderungen
- Er prüft ob es funktioniert

Der Agent nutzt dafür ein **Large Language Model (LLM)** wie ChatGPT oder Google Gemini, kombiniert mit **Tools** die ihm erlauben, Dateien zu lesen, zu ändern und Git-Befehle auszuführen.

---

## 2. Die Grundidee (Konzept)

### Warum braucht ein LLM Tools?

Ein LLM wie ChatGPT kann nur Text generieren. Es kann nicht:
- Dateien auf deinem Computer lesen
- Code ausführen
- Git-Befehle ausführen

**Lösung:** Wir geben dem LLM "Tools" (Funktionen), die es aufrufen kann.

### Der Agent-Loop

```mermaid
flowchart TD
    A[🚀 START] --> B[User gibt Aufgabe ein]
    B --> C[System-Prompt wird erstellt]
    C --> D{LLM wird gefragt}
    D --> E[LLM antwortet mit Tool-Calls]
    E --> F[Tool wird ausgeführt]
    F --> G[Ergebnis geht zurück ans LLM]
    G --> H{Task fertig?}
    H -->|Nein| D
    H -->|Ja| I[✅ ENDE]

    style A fill:#4CAF50,color:#fff
    style I fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
    style H fill:#FF9800,color:#fff
```

### Vereinfachter Ablauf

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant A as 🤖 Agent
    participant L as 🧠 LLM (GPT/Gemini)
    participant T as 🔧 Tools

    U->>A: "Fix den Bug in login.py"

    loop Bis Task fertig
        A->>L: Conversation + Tools senden
        L->>A: Antwort + Tool-Calls
        A->>T: Tool ausführen (z.B. read_file)
        T->>A: Ergebnis
        A->>L: Ergebnis mitteilen
    end

    L->>A: "Task completed"
    A->>U: Ergebnis anzeigen
```

---

## 3. Architektur-Überblick

Das Projekt besteht aus **drei Schichten**:

```mermaid
flowchart TB
    subgraph Frontend["🖥️ FRONTEND"]
        ST[Streamlit UI<br/>localhost:8501]
        SW[Swagger UI<br/>localhost:8000/docs]
    end

    subgraph Backend["⚙️ BACKEND"]
        API[FastAPI Server<br/>api.py]
        DEV[DevAgent<br/>dev_agent.py]
    end

    subgraph Common["📦 COMMON"]
        LLM[LLM Client<br/>llm_client.py]
        MCP[MCP Client<br/>mcp_client.py]
        LOC[Local Tools<br/>local_tools.py]
        CON[Conversation<br/>conversation.py]
        SET[Settings<br/>settings.py]
        DIF[Unified Diff<br/>unified_diff.py]
    end

    subgraph External["🌐 EXTERNE SERVICES"]
        OAI[OpenAI API]
        GEM[Google Gemini API]
        GIT[mcp-server-git]
    end

    ST --> API
    SW --> API
    API --> DEV
    DEV --> LLM
    DEV --> MCP
    DEV --> LOC
    DEV --> CON
    LLM --> OAI
    LLM --> GEM
    MCP --> GIT
    LOC --> DIF
    LLM --> SET

    style Frontend fill:#E3F2FD
    style Backend fill:#FFF3E0
    style Common fill:#E8F5E9
    style External fill:#FCE4EC
```

---

## 4. Ordnerstruktur

```mermaid
flowchart LR
    subgraph Root["📁 devops_llm_2"]
        ENV[".env"]
        ENVYML["environment.yml"]
        README["README.md"]

        subgraph Scripts["📁 scripts"]
            TA["test_agent.py"]
            TL["test_llm.py"]
            TLT["test_local_tools.py"]
            TMG["test_mcp_git.py"]
        end

        subgraph Rusty["📁 rusty_2"]
            subgraph BE["📁 backend"]
                API2["api.py"]
                DA["dev_agent.py"]
                LT["local_tools.py"]
                subgraph Eval["📁 eval"]
                    ME["metrics.py"]
                    RE["run_eval.py"]
                end
            end

            subgraph CO["📁 common"]
                CV["conversation.py"]
                LC["llm_client.py"]
                MC["mcp_client.py"]
                MS["messages.py"]
                SS["settings.py"]
                UD["unified_diff.py"]
            end

            subgraph FE["📁 frontend"]
                AP["app.py"]
                SD["streamlit_display.py"]
            end
        end
    end

    style Root fill:#FAFAFA
    style Scripts fill:#E3F2FD
    style Rusty fill:#FFF8E1
    style BE fill:#FFECB3
    style CO fill:#C8E6C9
    style FE fill:#B3E5FC
    style Eval fill:#FFE0B2
```

### Datei-Übersicht

| Datei | Beschreibung |
|-------|--------------|
| `.env` | API Keys (NICHT in Git!) |
| `environment.yml` | Conda Dependencies |
| `api.py` | FastAPI Server |
| `dev_agent.py` | Kern-Agent-Logik |
| `local_tools.py` | read_file, apply_diff |
| `llm_client.py` | OpenAI/Gemini Abstraktion |
| `mcp_client.py` | Git-Tools via MCP |
| `conversation.py` | Chat-Verlauf Management |
| `app.py` | Streamlit Frontend |

---

## 5. Detaillierte Datei-Erklärungen

### 5.1 Backend-Dateien

---

#### `api.py` - Der FastAPI Server

**Pfad:** `rusty_2/backend/api.py`

**Was macht diese Datei?**
Sie stellt eine REST-API bereit, über die der Agent gestartet werden kann.

```mermaid
flowchart LR
    subgraph Endpoints
        E1["GET /"]
        E2["GET /health"]
        E3["POST /dev-agent/run"]
    end

    E1 --> R1["API Info"]
    E2 --> R2["Status Check"]
    E3 --> R3["Agent starten"]

    style E3 fill:#4CAF50,color:#fff
```

**Request-Struktur:**

```python
class DevAgentRunRequest:
    task_description: str      # Was soll der Agent tun?
    repo_root: str             # Pfad zum Repository
    git_mcp_url: str           # URL für den Git MCP Server
    max_steps: int = 20        # Maximale Anzahl Schritte
```

---

#### `dev_agent.py` - Das Herzstück

**Pfad:** `rusty_2/backend/dev_agent.py`

```mermaid
classDiagram
    class DevAgentConfig {
        +int max_steps
        +str backend_name
        +str git_mcp_url
        +set allowed_tools
    }

    class DevAgentResult {
        +bool success
        +int steps
        +Conversation conversation
        +str error
    }

    class DevAgent {
        +ModelClient model_client
        +MCPToolClient mcp_tool_client
        +DevAgentConfig config
        +run_step(conversation)
        -_get_available_tools()
    }

    DevAgent --> DevAgentConfig
    DevAgent --> DevAgentResult
```

**Der run_step() Ablauf:**

```mermaid
flowchart TD
    A[run_step Start] --> B[Tools vom MCP holen]
    B --> C[LLM aufrufen mit<br/>Conversation + Tools]
    C --> D[Antwort zur<br/>Conversation hinzufügen]
    D --> E{Tool-Calls<br/>vorhanden?}
    E -->|Ja| F[Für jeden Tool-Call]
    F --> G{MCP oder<br/>Local Tool?}
    G -->|MCP| H[mcp_tool_client.call_tool]
    G -->|Local| I[local_tools.call_tool]
    H --> J[Ergebnis zur<br/>Conversation]
    I --> J
    J --> F
    E -->|Nein| K[run_step Ende]
    F -->|Alle fertig| K

    style A fill:#4CAF50,color:#fff
    style K fill:#4CAF50,color:#fff
```

---

#### `local_tools.py` - Datei-Operationen

**Verfügbare Tools:**

```mermaid
flowchart LR
    subgraph LocalTools["🔧 Local Tools"]
        RF["read_file"]
        AD["apply_unified_diff"]
    end

    RF --> |"path"| FILE["📄 Datei lesen"]
    AD --> |"path + diff"| PATCH["✏️ Patch anwenden"]

    style LocalTools fill:#E8F5E9
```

**Sicherheits-Check:**

```mermaid
flowchart TD
    A["Pfad: ../../../etc/passwd"] --> B{Innerhalb<br/>repo_root?}
    B -->|Nein| C["❌ ValueError"]
    B -->|Ja| D["✅ Zugriff erlaubt"]

    style C fill:#f44336,color:#fff
    style D fill:#4CAF50,color:#fff
```

---

### 5.2 Common-Dateien

---

#### `llm_client.py` - LLM Kommunikation

```mermaid
flowchart TB
    subgraph ModelClient
        GEN["generate()"]
    end

    subgraph Backends["Unterstützte Backends"]
        OAI["OpenAI<br/>gpt-4o-mini"]
        GEM["Gemini<br/>gemini-pro"]
    end

    GEN --> |"OPENAI_API_KEY"| OAI
    GEN --> |"GOOGLE_API_KEY"| GEM

    OAI --> API1["api.openai.com"]
    GEM --> API2["generativelanguage<br/>.googleapis.com"]

    style OAI fill:#10A37F,color:#fff
    style GEM fill:#4285F4,color:#fff
```

**Rate-Limiting:**

```mermaid
sequenceDiagram
    participant C as Client
    participant RL as Rate Limiter
    participant API as LLM API

    C->>RL: Request 1
    RL->>API: ✅ Sofort senden
    API->>C: Response

    C->>RL: Request 2 (nach 0.5s)
    RL->>RL: ⏳ Warte 0.5s
    RL->>API: Senden
    API->>C: Response
```

---

#### `mcp_client.py` - Git-Tool Anbindung

**Was ist MCP?**

```mermaid
flowchart LR
    subgraph Agent
        DA["DevAgent"]
    end

    subgraph MCP["MCP Protocol"]
        MC["MCPToolClient"]
    end

    subgraph Server["mcp-server-git"]
        GS["git_status"]
        GL["git_log"]
        GD["git_diff"]
        GB["git_branch"]
    end

    DA <--> |"stdio://"| MC
    MC <--> Server

    style MCP fill:#9C27B0,color:#fff
```

**Verfügbare Git-Tools:**

| Tool | Beschreibung |
|------|--------------|
| `git_status` | Zeigt geänderte Dateien |
| `git_log` | Zeigt Commit-Historie |
| `git_diff` | Zeigt Änderungen |
| `git_show` | Zeigt einen Commit |
| `git_branch_list` | Listet Branches |

---

#### `conversation.py` - Chat-Verlauf

```mermaid
flowchart TD
    subgraph Conversation
        M1["system: Du bist ein Assistent..."]
        M2["user: Fix den Bug"]
        M3["assistant: Ich lese die Datei..."]
        M4["tool: Dateiinhalt: def login()..."]
        M5["assistant: Task completed"]
    end

    M1 --> M2 --> M3 --> M4 --> M5

    style M1 fill:#9E9E9E,color:#fff
    style M2 fill:#2196F3,color:#fff
    style M3 fill:#4CAF50,color:#fff
    style M4 fill:#FF9800,color:#fff
    style M5 fill:#4CAF50,color:#fff
```

**Message-Rollen:**

| Rolle | Farbe | Bedeutung |
|-------|-------|-----------|
| `system` | 🔘 Grau | Anweisungen für das LLM |
| `user` | 🔵 Blau | Nachrichten vom Benutzer |
| `assistant` | 🟢 Grün | Antworten vom LLM |
| `tool` | 🟠 Orange | Ergebnisse von Tool-Aufrufen |

---

#### `unified_diff.py` - Code-Änderungen

**Was ist ein Unified Diff?**

```diff
--- original.py
+++ modified.py
@@ -10,7 +10,8 @@
 def calculate_total(items):
     total = 0
     for item in items:
-        total += item.price
+        total += item.price * item.quantity
+    total = round(total, 2)
     return total
```

```mermaid
flowchart LR
    subgraph Diff["Unified Diff"]
        H["Header<br/>--- / +++"]
        HU["Hunk Header<br/>@@ -10,7 +10,8 @@"]
        L["Lines<br/>- / + / space"]
    end

    H --> HU --> L

    L --> |"-"| DEL["🔴 Entfernt"]
    L --> |"+"| ADD["🟢 Hinzugefügt"]
    L --> |" "| CTX["⚪ Kontext"]
```

---

### 5.3 Frontend-Dateien

---

#### `app.py` - Streamlit Hauptanwendung

```mermaid
flowchart TB
    subgraph UI["Streamlit UI"]
        subgraph Sidebar
            S1["📁 Repo root"]
            S2["🔗 Git MCP URL"]
            S3["🔢 Max steps"]
            S4["🌐 API URL"]
        end

        subgraph Main
            M1["📝 Task Description"]
            M2["▶️ Run Button"]
            M3["📊 Results"]
            M4["💬 Conversation"]
        end
    end

    M2 --> |"HTTP POST"| API["FastAPI<br/>:8000"]
    API --> M3
    API --> M4

    style M2 fill:#4CAF50,color:#fff
```

---

### 5.4 Evaluations-Framework

```mermaid
flowchart TD
    subgraph Input
        TY["tasks.yaml"]
    end

    subgraph Eval["run_eval.py"]
        E1["Agent ausführen"]
        E2["pytest --collect-only"]
        E3["pytest"]
        E4["ruff check"]
    end

    subgraph Output
        CSV["eval_summary.csv"]
        JSON["eval_summary.json"]
    end

    TY --> E1
    E1 --> E2
    E2 --> |"success_compile"| CSV
    E1 --> E3
    E3 --> |"success_tests"| CSV
    E1 --> E4
    E4 --> |"success_static"| CSV
    CSV --> JSON

    style Input fill:#E3F2FD
    style Output fill:#E8F5E9
```

**Metriken:**

| Metrik | Beschreibung | Check |
|--------|--------------|-------|
| `success_compile` | Kompiliert? | `pytest --collect-only` |
| `success_tests` | Tests grün? | `pytest` |
| `success_static` | Linter OK? | `ruff` / `flake8` |
| `success_behaviour` | Aufgabe gelöst? | Tests bestanden |
| `steps` | Schritte | Zähler |

---

## 6. Verwendete Pakete

```mermaid
flowchart TB
    subgraph Core["🐍 Core"]
        PY["Python 3.11"]
    end

    subgraph Web["🌐 Web"]
        FA["FastAPI"]
        UV["Uvicorn"]
        ST["Streamlit"]
        RQ["Requests"]
    end

    subgraph AI["🤖 AI/LLM"]
        OA["OpenAI"]
        MC["MCP"]
        MG["mcp-server-git"]
    end

    subgraph Utils["🔧 Utils"]
        PD["Pydantic"]
        DE["python-dotenv"]
        YA["PyYAML"]
    end

    PY --> Web
    PY --> AI
    PY --> Utils

    style Core fill:#306998,color:#fff
    style Web fill:#009688,color:#fff
    style AI fill:#9C27B0,color:#fff
    style Utils fill:#FF9800,color:#fff
```

---

## 7. Kompletter Datenfluss

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant ST as 🖥️ Streamlit
    participant API as ⚙️ FastAPI
    participant AG as 🤖 DevAgent
    participant LLM as 🧠 LLM
    participant MCP as 🔧 MCP/Tools

    U->>ST: Task eingeben + Run klicken
    ST->>API: POST /dev-agent/run
    API->>AG: run_task()

    rect rgb(230, 245, 255)
        note over AG,MCP: Agent Loop
        AG->>LLM: Conversation + Tools
        LLM->>AG: Response + Tool-Calls
        AG->>MCP: Tool ausführen
        MCP->>AG: Ergebnis
        AG->>AG: Conversation updaten
    end

    AG->>API: DevAgentResult
    API->>ST: JSON Response
    ST->>U: Ergebnis anzeigen
```

---

## 8. Glossar

| Begriff | Erklärung |
|---------|-----------|
| **LLM** | Large Language Model - KI-Modell das Text versteht und generiert |
| **API** | Application Programming Interface - Schnittstelle für Services |
| **MCP** | Model Context Protocol - Anthropic's Protokoll für Tool-Anbindung |
| **Tool-Calling** | Feature von LLMs, externe Funktionen aufzurufen |
| **Agent** | Autonomes System: LLM + Tools + Loop |
| **Conversation** | Chat-Verlauf zwischen User, LLM und Tools |
| **System-Prompt** | Anweisungen die dem LLM sagen, wie es sich verhalten soll |
| **Unified Diff** | Standard-Format für Code-Änderungen |
| **Rate-Limiting** | Begrenzung der API-Anfragen pro Zeiteinheit |
| **FastAPI** | Modernes Python Web-Framework |
| **Streamlit** | Python Framework für Web-UIs |
| **Conda** | Paket- und Environment-Manager |

---

## 9. Quick Reference

### Befehle

```bash
# Environment aktivieren
conda activate rusty_2

# Backend starten
uvicorn rusty_2.backend.api:app --reload

# Frontend starten
streamlit run rusty_2/frontend/app.py

# LLM-Verbindung testen
python scripts/test_llm.py

# Agent testen
python scripts/test_agent.py

# Evaluation durchführen
python -m rusty_2.backend.eval.run_eval --tasks tasks.yaml
```

### URLs

| URL | Beschreibung |
|-----|--------------|
| http://localhost:8000 | FastAPI Backend |
| http://localhost:8000/docs | Swagger UI |
| http://localhost:8501 | Streamlit Frontend |

### Wichtige Dateien

| Datei | Beschreibung |
|-------|--------------|
| `.env` | API Keys (nicht in Git) |
| `environment.yml` | Conda Dependencies |
| `tasks.yaml` | Evaluations-Tasks |

---

*Dokumentation erstellt für DevOps & LLMs - HSLU Master*
*Stand: Dezember 2024*
