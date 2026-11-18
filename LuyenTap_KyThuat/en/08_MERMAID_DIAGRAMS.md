# 8. Mermaid Diagrams (draw quickly when explaining)

Memorize a few of these so you can **redraw them on a whiteboard/paper** when asked "draw the architecture / data flow". Mermaid renders on GitHub and many editors.

## 1. Generic "client → API → service" (agent-talk / totp-manager / outlook)
```mermaid
flowchart LR
  U[Browser / Frontend] -->|HTTP JSON| API[FastAPI/Flask]
  API --> SVC[Service / logic layer]
  SVC --> EXT[(External services:\nLLM / Graph / IMAP)]
  SVC --> ST[(Storage:\nfile / vault / localStorage)]
```

## 2. Token-by-token streaming (agent-talk)
```mermaid
sequenceDiagram
  participant FE as Frontend
  participant API as FastAPI
  participant LLM
  FE->>API: POST /turn_stream
  API->>LLM: chat (stream=true)
  loop each chunk
    LLM-->>API: delta
    API-->>FE: yield delta (StreamingResponse)
    FE->>FE: append to chat bubble
  end
```

## 3. "No stored key" encryption (totp-manager)
```mermaid
flowchart LR
  PW[Master password] -->|scrypt + salt| KEY[32B key, RAM only]
  KEY -->|Fernet AES| ENC{{Encrypt}}
  ACC[Account list] --> ENC
  ENC --> FILE[(vault.bin = salt + ciphertext)]
```

## 4. Two-tier storage (quan-quen-map)
```mermaid
flowchart TB
  APP[App] -->|place text| LS[(localStorage:\nid + category)]
  APP -->|base64 images| IDB[(IndexedDB:\nid -> dataUrl)]
  IDB -->|startup: hydrate| CACHE[imageCache RAM]
  CACHE --> APP
```

## 5. Check pipeline (word-checker)
```mermaid
flowchart LR
  DOC[load_document\n.docx] --> R{run_checks}
  R --> F[formatting]
  R --> T[textcheck]
  R --> S[spelling]
  R --> H[headings]
  F & T & S & H --> ISS[(list of Issue)]
  ISS --> OUT[report / pdf / json / autofix / annotate]
```

## 6. Service Worker lifecycle (PWA — cap-cuu-101 / quan-quen-map)
```mermaid
stateDiagram-v2
  [*] --> install: register
  install --> activate: precache done (skipWaiting)
  activate --> fetch: clean old caches (clients.claim)
  fetch --> fetch: intercept request\n(cache-first / network-first / SWR)
```

## 7. Cancelable multi-threaded import job (outlook-mail-reader)
```mermaid
flowchart LR
  L[List of lines] --> Q[[Queue]]
  Q --> W1[Worker 1]
  Q --> W2[Worker 2]
  Q --> W3[Worker n ≤5]
  W1 & W2 & W3 -->|Lock-guarded update| JOB[(Job state:\nprocessed/added/failed)]
  JOB --> POLL[Frontend polls every 1.5s]
```

## Usage tips
- In a remote interview, you can type Mermaid into a shared editor.
- Offline (paper/whiteboard): remember **boxes + arrows for data flow**, syntax not needed.
- Minimal syntax to remember: `flowchart LR/TB`, `A --> B`, `A -->|label| B`, `[box]`, `[(cylinder = storage)]`, `{decision}`.
- `sequenceDiagram` for time-ordered flows; `stateDiagram-v2` for lifecycles/states.

> 💡 Being able to draw your project's architecture **from memory** is a big plus — it proves you truly understand the system you built.
