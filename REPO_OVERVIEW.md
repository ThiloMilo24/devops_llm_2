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

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           AGENT LOOP                                     │
│                                                                          │
│   ┌──────────┐                                                          │
│   │  START   │                                                          │
│   └────┬─────┘                                                          │
│        │                                                                │
│        ▼                                                                │
│   ┌──────────────────────────────────────────────┐                      │
│   │ 1. User gibt Aufgabe ein                     │                      │
│   │    "Fix den Bug in login.py"                 │                      │
│   └────────────────────┬─────────────────────────┘                      │
│                        │                                                │
│                        ▼                                                │
│   ┌──────────────────────────────────────────────┐                      │
│   │ 2. System-Prompt wird erstellt               │                      │
│   │    "Du bist ein Entwickler-Assistent..."     │                      │
│   └────────────────────┬─────────────────────────┘                      │
│                        │                                                │
│                        ▼                                                │
│   ┌──────────────────────────────────────────────┐      ┌────────────┐  │
│   │ 3. LLM wird gefragt                          │ ───▶ │  OpenAI /  │  │
│   │    "Was soll ich tun?"                       │ ◀─── │  Gemini    │  │
│   └────────────────────┬─────────────────────────┘      └────────────┘  │
│                        │                                                │
│                        ▼                                                │
│   ┌──────────────────────────────────────────────┐                      │
│   │ 4. LLM antwortet mit Tool-Calls              │                      │
│   │    "Ich rufe read_file('login.py') auf"      │                      │
│   └────────────────────┬─────────────────────────┘                      │
│                        │                                                │
│                        ▼                                                │
│   ┌──────────────────────────────────────────────┐      ┌────────────┐  │
│   │ 5. Tool wird ausgeführt                      │ ───▶ │  Dateien   │  │
│   │    Datei wird gelesen                        │ ◀─── │  Git, etc. │  │
│   └────────────────────┬─────────────────────────┘      └────────────┘  │
│                        │                                                │
│                        ▼                                                │
│   ┌──────────────────────────────────────────────┐                      │
│   │ 6. Ergebnis geht zurück ans LLM              │                      │
│   │    "Hier ist der Inhalt von login.py: ..."   │                      │
│   └────────────────────┬─────────────────────────┘                      │
│                        │                                                │
│                        ▼                                                │
│                 ┌──────────────┐                                        │
│                 │ Task fertig? │                                        │
│                 └──────┬───────┘                                        │
│                        │                                                │
│              ┌─────────┴─────────┐                                      │
│              │                   │                                      │
│              ▼                   ▼                                      │
│         ┌────────┐          ┌────────┐                                  │
│         │  NEIN  │          │   JA   │                                  │
│         │  ───▶  │          │        │                                  │
│         │ Zurück │          │  ENDE  │                                  │
│         │ zu 3.  │          │        │                                  │
│         └────────┘          └────────┘                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Architektur-Überblick

Das Projekt besteht aus **drei Schichten**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   SCHICHT 1: FRONTEND (Was der User sieht)                              │
│   ════════════════════════════════════════                              │
│                                                                          │
│   ┌─────────────────────────┐    ┌─────────────────────────────┐        │
│   │                         │    │                             │        │
│   │   STREAMLIT UI          │    │   SWAGGER UI                │        │
│   │   ──────────────        │    │   ──────────                │        │
│   │                         │    │                             │        │
│   │   - Schöne Oberfläche   │    │   - Auto-generiert          │        │
│   │   - Chat-Darstellung    │    │   - API-Dokumentation       │        │
│   │   - Für Endbenutzer     │    │   - Für Entwickler          │        │
│   │                         │    │                             │        │
│   │   Port: 8501            │    │   Port: 8000/docs           │        │
│   │   app.py                │    │   (von FastAPI)             │        │
│   │                         │    │                             │        │
│   └───────────┬─────────────┘    └──────────────┬──────────────┘        │
│               │                                 │                        │
│               └─────────────┬───────────────────┘                        │
│                             │ HTTP Requests                              │
│                             ▼                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   SCHICHT 2: BACKEND (API + Logik)                                      │
│   ════════════════════════════════                                      │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────┐       │
│   │                                                              │       │
│   │   FASTAPI SERVER (api.py)                                   │       │
│   │   ───────────────────────                                   │       │
│   │                                                              │       │
│   │   Endpoints:                                                 │       │
│   │   - POST /dev-agent/run  →  Startet den Agent               │       │
│   │   - GET  /health         →  Health-Check                    │       │
│   │   - GET  /               →  API-Info                        │       │
│   │                                                              │       │
│   └──────────────────────────┬──────────────────────────────────┘       │
│                              │                                           │
│                              ▼                                           │
│   ┌─────────────────────────────────────────────────────────────┐       │
│   │                                                              │       │
│   │   DEV AGENT (dev_agent.py)                                  │       │
│   │   ────────────────────────                                  │       │
│   │                                                              │       │
│   │   - Verwaltet die Conversation (Chat-Verlauf)               │       │
│   │   - Ruft das LLM auf                                        │       │
│   │   - Führt Tool-Calls aus                                    │       │
│   │   - Entscheidet wann Task fertig ist                        │       │
│   │                                                              │       │
│   └──────────────────────────┬──────────────────────────────────┘       │
│                              │                                           │
│               ┌──────────────┼──────────────┐                           │
│               │              │              │                           │
│               ▼              ▼              ▼                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   SCHICHT 3: COMMON (Gemeinsame Module)                                 │
│   ═════════════════════════════════════                                 │
│                                                                          │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               │
│   │               │  │               │  │               │               │
│   │  LLM CLIENT   │  │  MCP CLIENT   │  │ LOCAL TOOLS   │               │
│   │  ──────────   │  │  ──────────   │  │ ───────────   │               │
│   │               │  │               │  │               │               │
│   │  Spricht mit  │  │  Spricht mit  │  │  Datei-       │               │
│   │  OpenAI oder  │  │  Git MCP      │  │  Operationen  │               │
│   │  Gemini       │  │  Server       │  │               │               │
│   │               │  │               │  │  - read_file  │               │
│   │  llm_client   │  │  mcp_client   │  │  - apply_diff │               │
│   │  .py          │  │  .py          │  │               │               │
│   │               │  │               │  │  local_tools  │               │
│   └───────────────┘  └───────────────┘  │  .py          │               │
│                                         │               │               │
│                                         └───────────────┘               │
│                                                                          │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               │
│   │               │  │               │  │               │               │
│   │ CONVERSATION  │  │   SETTINGS    │  │ UNIFIED DIFF  │               │
│   │ ────────────  │  │   ────────    │  │ ────────────  │               │
│   │               │  │               │  │               │               │
│   │ Chat-Verlauf  │  │ Lädt .env     │  │ Parst und     │               │
│   │ speichern &   │  │ Datei mit     │  │ wendet Diff-  │               │
│   │ laden         │  │ API Keys      │  │ Patches an    │               │
│   │               │  │               │  │               │               │
│   │ conversation  │  │ settings.py   │  │ unified_diff  │               │
│   │ .py           │  │               │  │ .py           │               │
│   │               │  │               │  │               │               │
│   └───────────────┘  └───────────────┘  └───────────────┘               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Ordnerstruktur (Komplett)

```
devops_llm_2/                          # Repository Root
│
├── .env                               # API Keys (NICHT in Git!)
├── .gitignore                         # Git-Ignore Regeln
├── environment.yml                    # Conda Dependencies
├── README.md                          # Kurze Projekt-Beschreibung
├── REPO_OVERVIEW.md                   # Diese Datei
│
├── CICD/
│   └── Docker/
│       └── .gitkeep                   # Platzhalter (Docker noch nicht implementiert)
│
├── scripts/                           # Test- und Entwicklungs-Skripte
│   ├── test_agent.py                  # Testet den kompletten Agent
│   ├── test_llm.py                    # Testet die LLM-Verbindung
│   ├── test_local_tools.py            # Testet read_file und apply_diff
│   ├── test_mcp_git.py                # Testet den MCP Git Server
│   └── test_agent_local_edit.py       # Testet Agent mit lokalen Edits
│
└── rusty_2/                           # Hauptpaket
    ├── __init__.py                    # Macht rusty_2 zu einem Python-Paket
    │
    ├── backend/                       # Backend-Code
    │   ├── __init__.py
    │   ├── api.py                     # FastAPI Server
    │   ├── dev_agent.py               # Kern-Agent-Logik
    │   ├── local_tools.py             # Lokale Datei-Tools
    │   │
    │   └── eval/                      # Evaluations-Framework
    │       ├── __init__.py
    │       ├── metrics.py             # Evaluations-Metriken (EvalResult)
    │       ├── run_eval.py            # Führt Evaluationen durch
    │       └── tasks.yaml.example     # Beispiel für Task-Definition
    │
    ├── common/                        # Gemeinsam genutzte Module
    │   ├── __init__.py
    │   ├── conversation.py            # Chat-Verlauf Management
    │   ├── llm_client.py              # LLM API Abstraktion
    │   ├── mcp_client.py              # MCP Server Client
    │   ├── messages.py                # Message-Hilfsfunktionen
    │   ├── settings.py                # Konfiguration aus .env
    │   └── unified_diff.py            # Diff-Patch Parsing & Anwendung
    │
    └── frontend/                      # Frontend-Code
        ├── __init__.py
        ├── app.py                     # Streamlit Hauptanwendung
        └── streamlit_display.py       # Chat-Darstellungs-Komponenten
```

---

## 5. Detaillierte Datei-Erklärungen

### 5.1 Backend-Dateien

---

#### `api.py` - Der FastAPI Server

**Pfad:** `rusty_2/backend/api.py`

**Was macht diese Datei?**
Sie stellt eine REST-API bereit, über die der Agent gestartet werden kann. Das Streamlit-Frontend kommuniziert mit dieser API.

**Wichtige Teile erklärt:**

```python
# FastAPI App erstellen
app = FastAPI(
    title="DevAgent API",
    description="API for autonomous development assistant...",
    version="1.0.0",
)
```
→ Erstellt die API-Anwendung. FastAPI generiert automatisch eine Swagger-Dokumentation unter `/docs`.

```python
class DevAgentRunRequest(BaseModel):
    task_description: str      # Was soll der Agent tun?
    repo_root: str             # Pfad zum Repository
    git_mcp_url: str           # URL für den Git MCP Server
    max_steps: int = 20        # Maximale Anzahl Schritte
```
→ Definiert welche Daten der API-Request enthalten muss.

```python
@app.post("/dev-agent/run")
async def run_dev_agent(request: DevAgentRunRequest):
    # ... Agent wird gestartet ...
    result = await run_task(...)
    return DevAgentRunResponse(...)
```
→ Der Haupt-Endpoint. Wenn ein POST-Request kommt, wird der Agent gestartet.

**Endpoints:**

| Endpoint | Methode | Beschreibung | Beispiel |
|----------|---------|--------------|----------|
| `/` | GET | API-Info | `curl localhost:8000/` |
| `/health` | GET | Status-Check | `curl localhost:8000/health` |
| `/dev-agent/run` | POST | Agent starten | Siehe Swagger UI |

---

#### `dev_agent.py` - Das Herzstück

**Pfad:** `rusty_2/backend/dev_agent.py`

**Was macht diese Datei?**
Hier ist die gesamte Agent-Logik. Der Agent:
1. Erhält eine Aufgabe
2. Fragt das LLM was zu tun ist
3. Führt Tool-Calls aus
4. Wiederholt bis fertig

**Die wichtigsten Klassen:**

```python
@dataclass
class DevAgentConfig:
    max_steps: int = 20              # Nach 20 Schritten aufhören
    backend_name: str = "openai"     # Welches LLM? (openai oder gemini)
    git_mcp_url: str = ""            # URL für Git-Tools
    allowed_tools: Optional[set[str]] = None  # Welche Tools erlaubt?
```
→ Konfiguriert wie der Agent arbeitet.

```python
@dataclass
class DevAgentResult:
    success: bool                    # Hat es geklappt?
    steps: int                       # Wie viele Schritte gebraucht?
    conversation: Conversation       # Der komplette Chat-Verlauf
    error: Optional[str] = None      # Fehlermeldung falls nicht erfolgreich
```
→ Das Ergebnis nachdem der Agent fertig ist.

```python
class DevAgent:
    async def run_step(self, conversation: Conversation) -> None:
        # 1. Verfügbare Tools holen
        tools = await self._get_available_tools()

        # 2. LLM fragen
        response = await self.model_client.generate(
            messages=conversation.messages,
            tools=tools,
        )

        # 3. Antwort zur Conversation hinzufügen
        conversation.append(assistant_message_dict)

        # 4. Falls Tool-Calls: ausführen
        if tool_calls:
            for tool_call in tool_calls:
                # Tool ausführen (MCP oder lokal)
                tool_results = await self.mcp_tool_client.call_tool(...)
                # Ergebnis zur Conversation hinzufügen
                conversation.append(tool_message(...))
```
→ Ein einzelner Agent-Schritt.

**Wichtige Funktionen:**

```python
def create_initial_conversation(task_description: str, repo_root: str) -> Conversation:
```
→ Erstellt den System-Prompt. Dieser sagt dem LLM WER es ist und WAS es tun soll:

```
"You are an autonomous local development assistant.
You are working on a local repository located at: /path/to/repo

Your responsibilities:
- Use the available tools to inspect and modify files
- Run tests to verify changes work correctly
- Run static checks to ensure code quality
..."
```

```python
async def run_task(task_description, repo_root, config) -> DevAgentResult:
```
→ Führt die komplette Aufgabe aus. Ruft `run_step()` in einer Schleife auf bis:
- Das LLM sagt "Task completed"
- Oder `max_steps` erreicht ist

**Wann ist der Task fertig?**

```python
completion_phrases = [
    "i have completed the task",
    "task completed",
    "task is complete",
    "i have finished",
    "task finished",
    "completed the task",
]

if any(phrase in content for phrase in completion_phrases):
    return DevAgentResult(success=True, ...)
```
→ Der Agent schaut ob das LLM eine dieser Phrasen sagt.

---

#### `local_tools.py` - Datei-Operationen

**Pfad:** `rusty_2/backend/local_tools.py`

**Was macht diese Datei?**
Stellt dem Agent Tools zur Verfügung um Dateien zu lesen und zu ändern.

**Verfügbare Tools:**

| Tool | Beschreibung | Parameter |
|------|--------------|-----------|
| `read_file` | Liest eine Datei | `path`: Pfad zur Datei |
| `apply_unified_diff` | Wendet einen Diff-Patch an | `path`: Datei, `diff`: Der Patch |

**Tool-Definitionen (für das LLM):**

```python
LOCAL_TOOL_SPECS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read a UTF-8 text file from the repository.",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "Path to the file, relative to repo root."
                    }
                },
                "required": ["path"]
            }
        }
    },
    # ... apply_unified_diff ...
]
```
→ Diese Definitionen werden ans LLM geschickt, damit es weiss welche Tools existieren.

**Sicherheits-Check:**

```python
def _resolve_path(self, relative: str) -> Path:
    p = (self.repo_root / relative).resolve()
    if not str(p).startswith(str(self.repo_root.resolve())):
        raise ValueError(f"Path {relative!r} escapes the repository root")
    return p
```
→ Verhindert dass der Agent Dateien ausserhalb des Repos liest/ändert.

---

### 5.2 Common-Dateien (Gemeinsame Module)

---

#### `llm_client.py` - LLM Kommunikation

**Pfad:** `rusty_2/common/llm_client.py`

**Was macht diese Datei?**
Abstrahiert die Kommunikation mit verschiedenen LLM-Anbietern (OpenAI, Google Gemini).

**Warum Abstraktion?**
Damit man einfach zwischen OpenAI und Gemini wechseln kann, ohne den restlichen Code zu ändern.

**Unterstützte Backends:**

| Backend | API Endpoint | Default Modell | API Key Variable |
|---------|--------------|----------------|------------------|
| OpenAI | api.openai.com/v1 | gpt-4o-mini | OPENAI_API_KEY |
| Gemini | generativelanguage.googleapis.com | gemini-pro | GOOGLE_API_KEY |

**Wichtige Klassen:**

```python
@dataclass
class ModelConfig:
    backend: str           # "openai" oder "gemini"
    model_name: str        # z.B. "gpt-4o-mini"
    api_key: str           # Der API-Key
    base_url: str          # API-Endpoint URL
    requests_per_minute: int = 60  # Rate-Limiting
```

```python
class ModelClient:
    async def generate(self, messages, tools=None) -> dict:
        # Rate-Limiting einhalten
        # ...

        # API-Call machen
        response = await self._client.chat.completions.create(
            model=self.config.model_name,
            messages=normalized_messages,
            tools=tools,
        )

        return response.model_dump()
```

**Rate-Limiting erklärt:**

```python
self._min_interval = 60.0 / self._requests_per_minute  # z.B. 1 Sekunde

async def generate(...):
    if self._last_request_time is not None:
        elapsed = time.time() - self._last_request_time
        if elapsed < self._min_interval:
            await asyncio.sleep(self._min_interval - elapsed)  # Warten
```
→ Verhindert dass wir zu viele Requests schicken und geblockt werden.

---

#### `mcp_client.py` - Git-Tool Anbindung

**Pfad:** `rusty_2/common/mcp_client.py`

**Was macht diese Datei?**
Verbindet zum MCP (Model Context Protocol) Server, der Git-Tools bereitstellt.

**Was ist MCP?**
Ein Protokoll von Anthropic, um LLMs mit externen Tools zu verbinden. Wir nutzen `mcp-server-git`, einen MCP-Server der Git-Befehle als Tools bereitstellt.

**Wie funktioniert die Verbindung?**

```
┌─────────────┐     stdio      ┌─────────────────┐
│  DevAgent   │ ◀────────────▶ │  mcp-server-git │
│             │   (stdin/out)  │                 │
└─────────────┘                └─────────────────┘
```

Die URL sieht so aus:
```
stdio://python:-m:mcp_server_git:--repository:/pfad/zum/repo
```

**Verfügbare Git-Tools (vom MCP Server):**

| Tool | Beschreibung |
|------|--------------|
| `git_status` | Zeigt geänderte Dateien |
| `git_log` | Zeigt Commit-Historie |
| `git_diff` | Zeigt Änderungen |
| `git_show` | Zeigt einen Commit |
| `git_branch_list` | Listet Branches |
| ... | ... |

**Wichtige Methoden:**

```python
async def list_tools(self) -> list[dict]:
    """Holt alle verfügbaren Tools vom MCP Server"""

async def call_tool(self, name: str, arguments: dict) -> list[dict]:
    """Führt ein Tool aus und gibt das Ergebnis zurück"""
```

---

#### `conversation.py` - Chat-Verlauf

**Pfad:** `rusty_2/common/conversation.py`

**Was macht diese Datei?**
Verwaltet den Chat-Verlauf (alle Nachrichten zwischen User, Agent und Tools).

**Warum wichtig?**
- LLMs sind "stateless" - sie erinnern sich nicht an vorherige Nachrichten
- Wir müssen den kompletten Verlauf bei jedem API-Call mitschicken
- Diese Klasse macht das Management einfach

**Message-Struktur:**

```python
{
    "role": "user",           # Wer spricht? (system/user/assistant/tool)
    "content": "Fix den Bug", # Was wird gesagt?
    "tool_call_id": None      # Nur bei Tool-Messages
}
```

**Die verschiedenen Rollen:**

| Rolle | Bedeutung | Beispiel |
|-------|-----------|----------|
| `system` | Anweisungen für das LLM | "Du bist ein Entwickler-Assistent..." |
| `user` | Nachrichten vom Benutzer | "Fix den Bug in auth.py" |
| `assistant` | Antworten vom LLM | "Ich lese zuerst die Datei..." |
| `tool` | Ergebnisse von Tool-Aufrufen | "Dateiinhalt: def login()..." |

**Wichtige Methoden:**

```python
class Conversation:
    def append(self, *messages):
        """Fügt Nachrichten hinzu"""

    def save(self, path):
        """Speichert als JSON-Datei"""

    @classmethod
    def load(cls, path):
        """Lädt aus JSON-Datei"""

    def register_observer(self, observer):
        """Observer-Pattern für Live-Updates"""
```

**Observer-Pattern erklärt:**
```python
class MyObserver:
    def update(self, message):
        print(f"Neue Nachricht: {message}")

conv = Conversation()
conv.register_observer(MyObserver())
conv.append({"role": "user", "content": "Hallo"})
# Output: "Neue Nachricht: {'role': 'user', 'content': 'Hallo'}"
```
→ Nützlich um z.B. die UI live zu aktualisieren.

---

#### `unified_diff.py` - Code-Änderungen

**Pfad:** `rusty_2/common/unified_diff.py`

**Was macht diese Datei?**
Parst und wendet "Unified Diff" Patches an.

**Was ist ein Unified Diff?**
Ein Standard-Format um Code-Änderungen darzustellen:

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

**Erklärung:**
- `---` / `+++` : Welche Dateien
- `@@ -10,7 +10,8 @@` : Ab Zeile 10, 7 Zeilen alt → 8 Zeilen neu
- Zeilen mit `-` : Werden entfernt
- Zeilen mit `+` : Werden hinzugefügt
- Zeilen mit ` ` : Bleiben gleich (Kontext)

**Warum nutzen wir das?**
Das LLM kann nicht einfach "ändere Zeile 15". Stattdessen generiert es einen Diff-Patch, der dann angewendet wird. Das ist präziser und weniger fehleranfällig.

**Wichtige Klassen:**

```python
@dataclass
class UnifiedDiffHunk:
    """Ein einzelner Änderungsblock"""
    from_line: int      # Start-Zeile im Original
    from_count: int     # Anzahl Zeilen im Original
    to_line: int        # Start-Zeile im Ergebnis
    to_count: int       # Anzahl Zeilen im Ergebnis
    before: list[str]   # Original-Zeilen
    after: list[str]    # Neue Zeilen

@dataclass
class UnifiedDiff:
    """Ein kompletter Patch"""
    from_file: str              # Original-Datei
    to_file: str                # Ziel-Datei
    hunks: list[UnifiedDiffHunk]  # Alle Änderungsblöcke
```

---

#### `settings.py` - Konfiguration

**Pfad:** `rusty_2/common/settings.py`

**Was macht diese Datei?**
Lädt Konfiguration aus der `.env` Datei.

**Die `.env` Datei:**

```bash
# .env (im Repository-Root)
OPENAI_API_KEY=sk-proj-xxxxx
GOOGLE_API_KEY=AIzaSyxxxxx
LLM_BACKEND_NAME=openai        # oder "gemini"
LLM_MODEL_NAME=gpt-4o-mini     # oder "gemini-pro"
LLM_REQUESTS_PER_MINUTE=60
```

**Funktionen:**

```python
def load_env():
    """Lädt .env Datei"""

def get_google_api_key() -> str:
    """Holt GOOGLE_API_KEY (wirft Fehler wenn nicht gesetzt)"""

def get_openai_api_key() -> Optional[str]:
    """Holt OPENAI_API_KEY (oder None)"""

def get_default_backend_name() -> str:
    """Holt LLM_BACKEND_NAME (default: 'openai')"""

def get_default_model_name() -> str:
    """Holt LLM_MODEL_NAME (default: 'gpt-4o-mini')"""
```

---

### 5.3 Frontend-Dateien

---

#### `app.py` - Streamlit Hauptanwendung

**Pfad:** `rusty_2/frontend/app.py`

**Was macht diese Datei?**
Die Benutzeroberfläche, mit der man den Agent bedient.

**UI-Aufbau:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  🤖 Rusty 2.0 - Local Autonomous Development Assistant              │
├─────────────────────┬───────────────────────────────────────────────┤
│                     │                                               │
│  SIDEBAR            │  MAIN AREA                                    │
│  ────────           │  ─────────                                    │
│                     │                                               │
│  Configuration:     │  Task Description:                            │
│  ┌───────────────┐  │  ┌─────────────────────────────────────────┐  │
│  │ Repo root     │  │  │                                         │  │
│  │ /path/to/repo │  │  │  Describe what you want the agent       │  │
│  └───────────────┘  │  │  to do...                               │  │
│                     │  │                                         │  │
│  ┌───────────────┐  │  └─────────────────────────────────────────┘  │
│  │ Git MCP URL   │  │                                               │
│  │ stdio://...   │  │  ┌─────────────────────────────────────────┐  │
│  └───────────────┘  │  │         ▶ Run Dev Agent                 │  │
│                     │  └─────────────────────────────────────────┘  │
│  ┌───────────────┐  │                                               │
│  │ Max steps: 20 │  │  ─────────────────────────────────────────    │
│  └───────────────┘  │                                               │
│                     │  Execution Summary:                           │
│  ─────────────────  │  ┌──────────┬──────────┬──────────┐          │
│                     │  │ ✅ Success│ Steps: 5 │ No errors│          │
│  API Settings:      │  └──────────┴──────────┴──────────┘          │
│  ┌───────────────┐  │                                               │
│  │ localhost:8000│  │  Conversation:                                │
│  └───────────────┘  │  ┌─────────────────────────────────────────┐  │
│                     │  │ 💬 System: You are an assistant...      │  │
│                     │  │ 👤 You: Fix the bug...                  │  │
│                     │  │ 🤖 Assistant: I'll read the file...     │  │
│                     │  │ 🔧 Tool: [file content]                 │  │
│                     │  │ 🤖 Assistant: Task completed.           │  │
│                     │  └─────────────────────────────────────────┘  │
│                     │                                               │
└─────────────────────┴───────────────────────────────────────────────┘
```

**Wichtiger Code erklärt:**

```python
# Page configuration
st.set_page_config(
    page_title="Rust 2.0",
    page_icon="🤖",
    layout="wide",
)
```
→ Setzt Titel und Layout.

```python
# Sidebar
with st.sidebar:
    repo_root = st.text_input("Repo root path", ...)
    git_mcp_url = st.text_input("Git MCP URL", ...)
    max_steps = st.number_input("Max steps", ...)
```
→ Eingabefelder in der Sidebar.

```python
# Run button
if st.button("Run Dev Agent", type="primary"):
    with st.spinner("Running DevAgent..."):
        response = requests.post(
            "http://localhost:8000/dev-agent/run",
            json=payload,
        )
```
→ Wenn Button geklickt, wird API aufgerufen.

---

#### `streamlit_display.py` - Chat-Darstellung

**Pfad:** `rusty_2/frontend/streamlit_display.py`

**Was macht diese Datei?**
Rendert den Chat-Verlauf mit schönem Styling.

**Styling pro Rolle:**

| Rolle | Farbe | Position |
|-------|-------|----------|
| System | Grau mit blauem Rand | Links |
| User | Blau | Rechts |
| Assistant | Hellgrau mit grünem Rand | Links |
| Tool | Monospace Code-Block | Links |

---

### 5.4 Evaluations-Framework

---

#### `run_eval.py` - Automatische Evaluation

**Pfad:** `rusty_2/backend/eval/run_eval.py`

**Was macht diese Datei?**
Führt automatisch mehrere Tasks aus und misst wie gut der Agent ist.

**Workflow:**

```
┌─────────────────┐
│   tasks.yaml    │  ← Definiert welche Tasks getestet werden
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   run_eval.py   │  ← Für jeden Task:
│                 │     1. Agent ausführen
│                 │     2. Tests laufen lassen
│                 │     3. Linter laufen lassen
│                 │     4. Ergebnis speichern
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ eval_summary    │  ← CSV/JSON mit Ergebnissen
│ .csv / .json    │
└─────────────────┘
```

**Task-Definition (tasks.yaml):**

```yaml
- id: fix-login-bug
  description: "Fix the login validation bug in auth.py"
  repo_root: "/path/to/target/repo"
  git_mcp_url: "stdio://python:-m:mcp_server_git:--repository:/path/to/target/repo"

- id: add-unit-test
  description: "Add a unit test for calculate_total function"
  repo_root: "/path/to/target/repo"
  git_mcp_url: "stdio://python:-m:mcp_server_git:--repository:/path/to/target/repo"
```

**Evaluations-Metriken:**

| Metrik | Beschreibung | Wie gemessen |
|--------|--------------|--------------|
| `success_compile` | Kompiliert der Code? | `pytest --collect-only` |
| `success_tests` | Bestehen Unit Tests? | `pytest` |
| `success_static` | Keine Linter-Fehler? | `ruff check` oder `flake8` |
| `success_behaviour` | Aufgabe gelöst? | Tests bestanden |
| `steps` | Wie viele Schritte? | Zähler |

**Aufruf:**

```bash
python -m rusty_2.backend.eval.run_eval \
    --tasks tasks.yaml \
    --output-dir ./eval_results \
    --max-steps 20
```

---

### 5.5 Test-Skripte

---

#### `test_llm.py` - LLM-Verbindung testen

**Pfad:** `scripts/test_llm.py`

**Was macht es?**
Testet ob die Verbindung zum LLM funktioniert.

**Aufruf:**
```bash
python scripts/test_llm.py
```

**Erwartete Ausgabe:**
```
LLM reply: Hello! How can I help you today?
```

---

#### `test_agent.py` - Agent testen

**Pfad:** `scripts/test_agent.py`

**Was macht es?**
Führt einen kompletten Agent-Durchlauf mit einer Test-Aufgabe aus.

**Test-Task:**
```python
task = "Inspect this repository and briefly describe what the test files do."
```

**Aufruf:**
```bash
python scripts/test_agent.py
```

---

## 6. Verwendete Pakete

| Paket | Version | Beschreibung | Wo verwendet |
|-------|---------|--------------|--------------|
| `python` | 3.11 | Programmiersprache | Überall |
| `fastapi` | * | Modernes Web-Framework für APIs | api.py |
| `uvicorn` | * | ASGI Server (führt FastAPI aus) | Terminal |
| `streamlit` | * | Framework für Data-Science UIs | frontend/ |
| `openai` | * | Offizielle OpenAI Python Library | llm_client.py |
| `mcp` | * | Model Context Protocol Library | mcp_client.py |
| `mcp-server-git` | * | Git-Tools als MCP Server | mcp_client.py |
| `requests` | * | HTTP Client | app.py |
| `python-dotenv` | * | .env Datei laden | settings.py |
| `pyyaml` | * | YAML Parser | run_eval.py |
| `pydantic` | * | Datenvalidierung | api.py |

---

## 7. Datenfluss (Komplett)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. USER EINGABE                                                         │
│  ────────────────                                                        │
│                                                                          │
│  User öffnet Streamlit (localhost:8501)                                  │
│  User gibt ein: "Fix the bug in login.py"                               │
│  User klickt: "Run Dev Agent"                                           │
│                                                                          │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  2. STREAMLIT → FASTAPI                                                  │
│  ──────────────────────                                                  │
│                                                                          │
│  HTTP POST Request an localhost:8000/dev-agent/run                       │
│  Body: {                                                                 │
│    "task_description": "Fix the bug in login.py",                       │
│    "repo_root": "/path/to/repo",                                        │
│    "git_mcp_url": "stdio://python:-m:mcp_server_git:...",              │
│    "max_steps": 20                                                       │
│  }                                                                       │
│                                                                          │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  3. FASTAPI → DEV AGENT                                                  │
│  ──────────────────────                                                  │
│                                                                          │
│  api.py ruft run_task() auf                                             │
│  DevAgentConfig wird erstellt                                            │
│  ModelClient wird erstellt (für LLM)                                     │
│  MCPToolClient wird erstellt (für Git-Tools)                            │
│  Initiale Conversation wird erstellt (System-Prompt + User-Message)     │
│                                                                          │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  4. AGENT LOOP (wiederholt sich bis fertig)                             │
│  ──────────────────────────────────────────                             │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  4a. LLM ANFRAGE                                                   │ │
│  │  ─────────────────                                                 │ │
│  │                                                                    │ │
│  │  DevAgent sendet Conversation + Tool-Definitionen an LLM          │ │
│  │  → OpenAI API (api.openai.com) oder                               │ │
│  │  → Gemini API (generativelanguage.googleapis.com)                 │ │
│  │                                                                    │ │
│  └────────────────────────────────────┬───────────────────────────────┘ │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  4b. LLM ANTWORT                                                   │ │
│  │  ───────────────                                                   │ │
│  │                                                                    │ │
│  │  LLM antwortet mit Text UND/ODER Tool-Calls:                      │ │
│  │  {                                                                 │ │
│  │    "content": "I'll read the file first.",                        │ │
│  │    "tool_calls": [{                                                │ │
│  │      "function": {                                                 │ │
│  │        "name": "read_file",                                        │ │
│  │        "arguments": "{\"path\": \"login.py\"}"                    │ │
│  │      }                                                             │ │
│  │    }]                                                              │ │
│  │  }                                                                 │ │
│  │                                                                    │ │
│  └────────────────────────────────────┬───────────────────────────────┘ │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  4c. TOOL AUSFÜHRUNG                                               │ │
│  │  ───────────────────                                               │ │
│  │                                                                    │ │
│  │  Für jeden Tool-Call:                                              │ │
│  │  - Ist es ein MCP-Tool (git_*)? → MCPToolClient.call_tool()       │ │
│  │  - Ist es ein lokales Tool? → LocalToolExecutor.call_tool()       │ │
│  │                                                                    │ │
│  │  Ergebnis wird zur Conversation hinzugefügt                       │ │
│  │                                                                    │ │
│  └────────────────────────────────────┬───────────────────────────────┘ │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  4d. CHECK: TASK FERTIG?                                           │ │
│  │  ───────────────────────                                           │ │
│  │                                                                    │ │
│  │  Enthält die letzte Assistant-Message "task completed"?           │ │
│  │  - JA → Loop beenden, Ergebnis zurückgeben                        │ │
│  │  - NEIN → Weiter bei 4a (nächster Schritt)                        │ │
│  │                                                                    │ │
│  │  Ist max_steps erreicht?                                           │ │
│  │  - JA → Loop beenden, success=False                               │ │
│  │  - NEIN → Weiter bei 4a                                           │ │
│  │                                                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  5. ERGEBNIS ZURÜCK                                                      │
│  ──────────────────                                                      │
│                                                                          │
│  DevAgentResult wird erstellt:                                           │
│  - success: True/False                                                   │
│  - steps: Anzahl Schritte                                                │
│  - conversation: Kompletter Chat-Verlauf                                 │
│  - error: Fehlermeldung (falls vorhanden)                               │
│                                                                          │
│  FastAPI wandelt in JSON um und sendet Response                         │
│  Streamlit zeigt Ergebnis an                                            │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Glossar

| Begriff | Erklärung |
|---------|-----------|
| **LLM** | Large Language Model - KI-Modell das Text versteht und generiert (z.B. GPT-4, Gemini) |
| **API** | Application Programming Interface - Schnittstelle um mit einem Service zu kommunizieren |
| **REST API** | Ein API-Stil der HTTP-Methoden (GET, POST, etc.) nutzt |
| **Endpoint** | Eine URL die eine bestimmte Funktion bereitstellt (z.B. `/dev-agent/run`) |
| **MCP** | Model Context Protocol - Anthropic's Protokoll um LLMs mit Tools zu verbinden |
| **Tool-Calling** | Feature von LLMs, externe Funktionen aufzurufen |
| **Agent** | Autonomes System das Aufgaben selbstständig bearbeitet (LLM + Tools + Loop) |
| **Conversation** | Der Chat-Verlauf zwischen User, LLM und Tools |
| **System-Prompt** | Anweisungen die dem LLM sagen, wie es sich verhalten soll |
| **Unified Diff** | Standard-Format für Code-Änderungen |
| **Rate-Limiting** | Begrenzung der Anzahl API-Anfragen pro Zeiteinheit |
| **FastAPI** | Modernes Python Web-Framework für APIs |
| **Streamlit** | Python Framework für interaktive Web-UIs |
| **ASGI** | Async Server Gateway Interface - Standard für async Python Web-Apps |
| **Uvicorn** | ASGI Server der FastAPI-Apps ausführt |
| **Conda** | Paket- und Environment-Manager für Python |
| **Environment** | Isolierte Python-Installation mit eigenen Paketen |
| **dotenv** | Technik um Konfiguration aus `.env` Dateien zu laden |

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
| http://localhost:8000/docs | Swagger UI (API-Dokumentation) |
| http://localhost:8501 | Streamlit Frontend |

### Dateien

| Datei | Beschreibung |
|-------|--------------|
| `.env` | API Keys (nicht in Git) |
| `environment.yml` | Conda Dependencies |
| `tasks.yaml` | Evaluations-Tasks |

---

*Dokumentation erstellt für DevOps & LLMs - HSLU Master*
*Stand: Dezember 2024*
