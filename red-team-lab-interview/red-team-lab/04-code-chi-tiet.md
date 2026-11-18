# 04 — Code chi tiết

| File | Vai trò |
|---|---|
| `levels.js` | `LEVELS` (7 màn direct), bí mật giả, cấu hình phòng thủ, gợi ý và bài học. |
| `engine.js` | `TECHNIQUES` nhận diện pattern; `runAttack(level, userText, state)` áp filter, judge, ngưỡng, biến đổi secret, trả kết quả/state; `checkGuess`. |
| `indirect.js` | `TECHS`, `RAG_LEVELS` (5 màn); `runIndirect(level, poison)` xử lý tài liệu, provenance, output scan. |
| `app.js` | `route()`, `renderLevel`, `renderRag`, `renderSandbox`, progress/score UI. |
| `store.js` | `store.solved`, `markSolved`, `recordScore`, `reset` trên `localStorage`. |
| `llm.js` | `providerList`, `buildSystemPrompt`, `callLLM`, `detectLeak`; chỉ cho sandbox tùy chọn. |
| `tests/*.mjs` | Test Node cho engine, indirect và store; `npm test` chạy cả ba. |

Nhìn thứ tự các nhánh trong `runIndirect`: lọc chỉ thị → phát hiện kỹ thuật → provenance → ngưỡng → output scan. Đổi thứ tự có thể đổi kết quả bài học; unit test cần bám các trường hợp bị chặn và bị lộ.
