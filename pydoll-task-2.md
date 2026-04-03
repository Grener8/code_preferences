  # `pydoll-serve` — Production Specification

  ## Objective

  Implement a **stateful HTTP service** that exposes Pydoll as a remote execution engine.

  The service must allow external systems (LLMs, pipelines, non-Python clients) to:

  * create and manage browser sessions
  * perform human-like interactions (click, type, navigation)
  * extract structured data or HTML
  * maintain session continuity across multiple requests

  ---

  ## Architecture

  ### Runtime Model

  ```text
  - 1 browser instance per service process
  - Each session = 1 tab
  - Tabs persist across requests
  - Sessions mapped to tab + state container
  ```

  ---

  ### Concurrency Model

  * Global async event loop (`asyncio`)
  * Limit active tabs via:

  ```python
  SESSION_CONCURRENCY = N
  ```

  * Use:

  ```python
  asyncio.Semaphore(SESSION_CONCURRENCY)
  ```

  ---

  ### Session Model

  Each session represents:

  ```text
  - Tab instance
  - Cookie store
  - Navigation state
  - Optional metadata (headers, proxy, etc.)
  - Last activity timestamp
  ```

  ---

  ## CLI Interface

  ```bash
  pydoll serve --host 0.0.0.0 --port 8000
  ```

  ### Options

  ```bash
  --port
  --host
  --max-sessions
  --session-timeout
  --browser-options (JSON)
  --headless
  ```

  ---

  ## API Design (Stateful)

  ### 1. Create Session

  ```http
  POST /sessions
  ```

  #### Body

  ```json
  {
    "options": {
      "headless": true,
      "proxy": "...",
      "geolocation": "US"
    }
  }
  ```

  #### Response

  ```json
  {
    "session_id": "uuid"
  }
  ```

  ---

  ### 2. Navigate

  ```http
  POST /sessions/{id}/navigate
  ```

  #### Body

  ```json
  {
    "url": "https://example.com",
    "timeout": 15000,
    "wait_until": "load"
  }
  ```

  ---

  ### 3. Perform Actions

  ```http
  POST /sessions/{id}/actions
  ```

  #### Body

  ```json
  {
    "actions": [...]
  }
  ```

  ---

  ### Action Schema (Full Exposure)

  ```json
  {
    "type": "click | type | scroll | wait | evaluate | keypress",
    "selector": "string",
    "text": "string",
    "key": "Enter",
    "script": "JS code",
    "for": "selector | sleep",
    "value": "string",
    "ms": 1000
  }
  ```

  ---

  ### Execution Semantics

  * Sequential execution
  * Fail-fast by default
  * Optional:

  ```json
  { "continue_on_error": true }
  ```

  ---

  ## 4. Extract Content

  ```http
  GET /sessions/{id}/content
  ```

  #### Query Params

  ```text
  format=html | json
  ```

  ---

  ### HTML Response

  ```json
  {
    "content": "<html>...</html>"
  }
  ```

  ---

  ### JSON Extraction (Future-proofed)

  ```http
  POST /sessions/{id}/extract
  ```

  #### Body

  ```json
  {
    "model": {
      "fields": [
        {"name": "title", "selector": "h1"}
      ]
    }
  }
  ```

  Returns structured JSON.

  ---

  ## 5. Cookies

  ### Get Cookies

  ```http
  GET /sessions/{id}/cookies
  ```

  ### Set Cookies

  ```http
  POST /sessions/{id}/cookies
  ```

  ---

  ## 6. Close Session

  ```http
  DELETE /sessions/{id}
  ```

  ---

  ## Session Lifecycle

  ### Creation

  * Creates tab via:

  ```python
  tab = await browser.start()
  ```

  ### Storage

  * Store in:

  ```python
  sessions: Dict[str, Session]
  ```

  ---

  ### Expiration

  * Sessions auto-expire after inactivity:

  ```python
  SESSION_TIMEOUT = seconds
  ```

  * Background cleanup task:

  ```python
  async def cleanup_loop():
      while True:
          delete expired sessions
  ```

  ---

  ## Browser Integration

  ### Initialization

  ```python
  browser = Chrome(options=...)
  ```

  * Created once at startup
  * Shared across sessions

  ---

  ## Action Engine Mapping

  | API Action    | Pydoll                 |
  | ------------- | ---------------------- |
  | click         | `el.click()`           |
  | type          | `el.insert_text()`     |
  | scroll        | `tab.evaluate(JS)`     |
  | evaluate      | `tab.evaluate(script)` |
  | keypress      | `tab.keyboard.press()` |
  | wait selector | `tab.find()`           |
  | sleep         | `asyncio.sleep()`      |

  ---

  ## Error Handling

  ### Error Types

  | Error           | HTTP Code |
  | --------------- | --------- |
  | Invalid session | 404       |
  | Timeout         | 408       |
  | Action failure  | 422       |
  | Browser crash   | 500       |

  ---

  ### Error Response

  ```json
  {
    "error": "Selector not found",
    "type": "ElementNotFound",
    "session_id": "..."
  }
  ```

  ---

  ## Timeout Model

  * Per request:

  ```json
  { "timeout": 15000 }
  ```

  * Enforced globally for:

    * navigation
    * actions

  ---

  ## Resource Management

  ### Limits

  ```text
  - max sessions
  - max concurrent actions
  ```

  If exceeded:

  * return HTTP 429

  ---

  ## Logging

  Must include:

  * session lifecycle events
  * navigation logs
  * action execution
  * errors with context

  ---

  ## Security (Minimal Production Baseline)

  * Optional API key:

  ```http
  Authorization: Bearer <token>
  ```

  * CORS support configurable

  ---

  ## Extensibility

  Design must allow:

  * custom action handlers
  * plugin system for:

    * extraction
    * markdown export
    * network interception

  ---

  ## Deployment

  * Must support:

    * Docker
    * single-process async server

  Recommended stack:

  * FastAPI / Starlette
  * Uvicorn

  ---

  ## Non-Goals (MVP)

  * distributed browser clusters
  * persistent storage of sessions
  * full HAR replay API (future)

  ---

  ## Success Criteria

  * External system can:

    * create session
    * navigate multiple pages
    * interact dynamically
    * extract HTML/JSON
  * Sessions behave like real users (state preserved)
  * Multiple sessions run concurrently without blocking
  * Service remains stable under load

