# TCP.Messaging.Server

> ℹ️ **This README was created with the support of AI** (Claude, based on the actual source code) — not exclusively AI-generated — and was reviewed by the author. Please double-check details before relying on them.

WebSocket relay server for a HoloLens/Unity application with human and AI assistance.
The server receives requests from a Unity client and forwards them — depending on the
selected mode — either to a human operator (browser) or to an AI backend. Answers are
sent back to the Unity client. In addition, study-relevant metrics (latencies,
questions/answers) are logged per session as JSONL. Core technology: Node.js with the
`ws` library.

> Developed and tested exclusively on **Windows** (Windows 11). This guide covers
> Windows/PowerShell only.

---

## Requirements

| Requirement | Details |
|---|---|
| Operating system | **Windows** (developed and tested on Windows 11). |
| Node.js | Tested with **v24.14.0**. Download: <https://nodejs.org/en/download> |
| npm | Tested with **11.9.0** (installed together with Node.js). |
| AI backend *(optional)* | Only needed for the `AiAssistance` mode: an HTTP service at `http://localhost:8000/ask` (see [External service](#external-service-ai-backend)). Not required for `HumanAssistance` alone. |
| Clients | A Unity/HoloLens client and/or a browser client that connect via WebSocket. Not part of this repo. |

**No** API keys, accounts, or environment variables are required to start the server.

### Network / firewall

The server listens on all network interfaces (`0.0.0.0`) on port **40002**. For clients
on the local network (e.g. a HoloLens) to connect, Windows must allow access to this
port. On first start, the Windows Firewall may prompt — allow it for private networks.

---

## Setup

1. Clone the repository (replace the URL with your target repo):

```powershell
git clone <your-repo-url>
```

2. Change into the project directory:

```powershell
cd <repo-folder>
```

3. Install the dependencies (reads [package.json](package.json) and installs the exact
   versions pinned in [package-lock.json](package-lock.json) locally into `node_modules\`):

```powershell
npm install
```

> **Note on isolation:** Unlike Python, no virtual environment is needed. `npm install`
> installs dependencies into the project-local `node_modules\` by default, which is
> already isolated from the rest of the system.

4. Check the configuration: there is no config file and no keys. Central settings
   (port, AI backend address) are defined as constants at the top of
   [server.js](server.js) — see [Configuration](#configuration).

---

## Start

```powershell
npm start
```

Or directly:

```powershell
node server.js
```

**What happens on first start:**

- The server generates a 9-digit session ID and prints it to the log.
- It lists all local IPv4 addresses it is reachable at, e.g.
  `Server reachable at: ws://192.168.x.x:40002`.
- The `metrics\` folder is created automatically on the first incoming request; one
  file `session_<id>.jsonl` is written per session.
- Every 10 seconds the server sends a `ping` heartbeat to connected clients.

**Reachability:** WebSocket at `ws://<host>:40002` (locally: `ws://localhost:40002`).

Stop with `Ctrl + C` — all open metrics files are closed cleanly.

---

## Try it out (quick test)

The server has no UI of its own. For manual testing, `wscat` (a WebSocket CLI) works
well. Install and connect:

```powershell
npm install -g wscat
```

```powershell
wscat -c ws://localhost:40002
```

After connecting, first register a role, then send a request (enter one JSON line each):

```json
{"type":"register","role":"unity"}
```

```json
{"type":"request","sessionId":"123456789","question":"What do you see here?","assistanceMode":"AiAssistance","language":"german"}
```

For `AiAssistance` the [AI backend](#external-service-ai-backend) must be running. For
`HumanAssistance` a second client must additionally be connected as
`{"type":"register","role":"browser"}`, which the request is forwarded to.

---

## Interface (WebSocket protocol)

No HTTP/REST — communication happens over WebSocket messages (JSON). Clients register
with a role (`unity` or `browser`). Exactly one active connection is kept per role.

| Message (type) | Direction | Purpose |
|---|---|---|
| `register` | Client → Server | Assign a connection to a role (`unity` / `browser`). |
| `session` | Server → Unity | Reply to `register` for role `unity`: passes the server session ID. |
| `request` | Unity → Server | Submit a request; routed to AI or browser depending on `assistanceMode`. |
| `response` | Browser → Server | Answer from a human operator; forwarded to Unity. |
| `ping` | Server → Client | Heartbeat every 10 s. |
| `pong` | Client → Server | Heartbeat reply (ignored/discarded). |

### Main path: `request`

Field overview:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `type` | string | yes | Must be `"request"`. |
| `assistanceMode` | string | yes | `"AiAssistance"` (AI backend) or `"HumanAssistance"` (browser). |
| `sessionId` | string | no | Session identifier. If missing, the server session ID is used. |
| `question` | string | no | The question/text. |
| `language` | string | no | Language for the AI backend (default `"german"`). |
| `image` | string (Base64) | no | Optional image. Can alternatively be sent as a binary frame (see below). |

**Binary frame (optional):** A `request` can also be sent as binary: 4 bytes
`UInt32BE` = length of the JSON part, then the JSON string, then the raw image bytes.
The server then appends the image as Base64 to the `image` field.

**Example request** (Unity → Server):

```json
{
  "type": "request",
  "sessionId": "123456789",
  "question": "What do you see here?",
  "assistanceMode": "AiAssistance",
  "language": "german"
}
```

**Example response** (Server → Unity, forwarded from the AI backend; the answer is
truncated to a maximum of 283 characters):

```json
{
  "session_id": "123456789",
  "answer": "There are two black iiyama computer monitors on a light wooden table.",
  "image_url": "",
  "audio_url": "",
  "boxes": []
}
```

### `response` (Browser → Unity)

Answer from a human operator. Important: here the field is called `session_id` (with
underscore), not `sessionId`.

| Field | Type | Meaning |
|---|---|---|
| `type` | string | Must be `"response"`. |
| `session_id` | string | Session identifier of the original request. |
| `answer` | string | Answer text (truncated to 283 characters). |
| `image_url` | string | Optional. |
| `audio_url` | string | Optional. |
| `boxes` | array | Optional (e.g. bounding boxes). |

### External service: AI backend

For `assistanceMode = "AiAssistance"` the server sends a `POST` as `multipart/form-data`
to `url` (default `http://localhost:8000/ask`):

| Form field | Content |
|---|---|
| `question` | Question text. |
| `session_id` | Session identifier. |
| `language` | Language (default `german`). |
| `image` | Optional: JPEG file (`image.jpg`) if an image was present. |

A JSON response with an `answer` field is expected. This backend is **not** part of this
repo and must be provided separately.

---

## Configuration

All central settings are defined as constants at the top of [server.js](server.js):

| Constant | Default | Meaning |
|---|---|---|
| `port` | `40002` | WebSocket port of the server. |
| `url` | `http://localhost:8000/ask` | Address of the AI backend. |
| `metrics` | `<project>\metrics` | Target folder for the session logs. |
| Heartbeat interval | `10000` ms | Interval of the `ping` messages (in `setInterval` at the end of the file). |

Restart the server after changes.

---

## Debugging / development

- **Logs:** The server writes detailed, timestamped logs to the console (connections,
  registrations, requests, forwarding, API calls, errors).
- **VS Code debugging:** Start the debugger on [server.js](server.js) in VS Code
  (Node.js configuration under "Run and Debug" → "Node.js"). Breakpoints can be set in
  the handler functions `handleAiAssistance` and `handleHumanAssistance`.
- **Manual testing:** see [Try it out (quick test)](#try-it-out-quick-test).

---

## Project structure

```
.
├─ server.js            # The WebSocket server (entry point)
├─ package.json         # Metadata, start script, dependencies
├─ package-lock.json    # Exact pinned dependency versions
├─ node_modules/        # Installed dependencies (via npm install)
├─ metrics/             # Session logs (JSONL), created at runtime
└─ README.md            # This guide
```

---

## Flow in brief

1. The server starts, generates a session ID, and listens on `ws://<host>:40002`.
2. A Unity client connects and registers with `{"type":"register","role":"unity"}`; it receives the session ID back.
3. Optionally, a browser client registers as `role: "browser"` (for the `HumanAssistance` mode).
4. Unity sends a `request` message with `assistanceMode` (AI or human), a question, and optionally an image.
5. The server logs the request as a metric and forwards it:
   - `AiAssistance` → `POST` to the AI backend (`url`), waits for the answer.
   - `HumanAssistance` → forwarded to the browser; the operator replies with `response`.
6. The server measures the latency (received → answered) and logs the answer.
7. The answer is sent to the Unity client (truncated to 283 characters if necessary).
8. On shutdown (`Ctrl + C`) all metrics files are closed cleanly.
