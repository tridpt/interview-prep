# 8. Sơ đồ Mermaid (vẽ nhanh khi giải thích)

Học thuộc vài sơ đồ này để **vẽ lại trên bảng/giấy** khi được hỏi "vẽ kiến trúc / luồng dữ liệu". Mermaid render được trên GitHub/nhiều editor.

## 1. Kiến trúc chung "client → API → service" (agent-talk / totp-manager / outlook)
```mermaid
flowchart LR
  U[Trình duyệt / Frontend] -->|HTTP JSON| API[FastAPI/Flask]
  API --> SVC[Lớp service / logic]
  SVC --> EXT[(Dịch vụ ngoài:\nLLM / Graph / IMAP)]
  SVC --> ST[(Lưu trữ:\nfile / vault / localStorage)]
```

## 2. Streaming từng chữ (agent-talk)
```mermaid
sequenceDiagram
  participant FE as Frontend
  participant API as FastAPI
  participant LLM
  FE->>API: POST /turn_stream
  API->>LLM: chat (stream=true)
  loop từng chunk
    LLM-->>API: delta
    API-->>FE: yield delta (StreamingResponse)
    FE->>FE: ghép vào bong bóng chat
  end
```

## 3. Mã hóa "không lưu khóa" (totp-manager)
```mermaid
flowchart LR
  PW[Mật khẩu chủ] -->|scrypt + salt| KEY[Khóa 32B chỉ ở RAM]
  KEY -->|Fernet AES| ENC{{Mã hóa}}
  ACC[Danh sách tài khoản] --> ENC
  ENC --> FILE[(vault.bin = salt + ciphertext)]
```

## 4. Lưu trữ hai tầng (quan-quen-map)
```mermaid
flowchart TB
  APP[App] -->|text quán| LS[(localStorage:\nid + category)]
  APP -->|ảnh base64| IDB[(IndexedDB:\nid -> dataUrl)]
  IDB -->|khởi động: hydrate| CACHE[imageCache RAM]
  CACHE --> APP
```

## 5. Pipeline kiểm tra (word-checker)
```mermaid
flowchart LR
  DOC[load_document\n.docx] --> R{run_checks}
  R --> F[formatting]
  R --> T[textcheck]
  R --> S[spelling]
  R --> H[headings]
  F & T & S & H --> ISS[(danh sách Issue)]
  ISS --> OUT[report / pdf / json / autofix / annotate]
```

## 6. Vòng đời Service Worker (PWA — cap-cuu-101 / quan-quen-map)
```mermaid
stateDiagram-v2
  [*] --> install: đăng ký
  install --> activate: precache xong (skipWaiting)
  activate --> fetch: dọn cache cũ (clients.claim)
  fetch --> fetch: chặn request\n(cache-first / network-first / SWR)
```

## 7. Job import đa luồng có hủy (outlook-mail-reader)
```mermaid
flowchart LR
  L[Danh sách dòng] --> Q[[Queue]]
  Q --> W1[Worker 1]
  Q --> W2[Worker 2]
  Q --> W3[Worker n ≤5]
  W1 & W2 & W3 -->|Lock cập nhật| JOB[(Trạng thái job:\nprocessed/added/failed)]
  JOB --> POLL[Frontend poll 1.5s]
```

## Mẹo dùng
- Khi phỏng vấn online, có thể gõ Mermaid vào một editor chia sẻ màn hình.
- Khi offline (giấy/bảng): nhớ **hình khối + mũi tên hướng dữ liệu**, không cần cú pháp.
- Cú pháp tối thiểu cần nhớ: `flowchart LR/TB`, `A --> B`, `A -->|nhãn| B`, `[hộp]`, `[(trụ = lưu trữ)]`, `{quyết định}`.
- `sequenceDiagram` cho luồng theo thời gian; `stateDiagram-v2` cho vòng đời/trạng thái.

> 💡 Vẽ được sơ đồ kiến trúc dự án của bạn **từ trí nhớ** là một điểm cộng lớn — nó chứng tỏ bạn thực sự hiểu hệ thống mình xây.
