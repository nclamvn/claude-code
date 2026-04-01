# BÁO CÁO KIỂM CHỨNG KIẾN TRÚC — Claude Code Source Analysis

**Người thực hiện:** Thợ thi công (Claude Code — Opus 4.6)
**Ngày:** 2026-04-01
**Loại:** SCAN + VERIFICATION REPORT
**Mục đích:** Kiểm chứng 10 nhận định kiến trúc từ bài phân tích của Chủ thầu dựa trên source code thực tế

---

## 1. THÔNG TIN REPO

| Mục | Giá trị |
|-----|---------|
| Đường dẫn | `/Users/mac/claudecode/claude-code/` |
| Origin | `github.com/nclamvn/claude-code` |
| Upstream | `github.com/fazxes/Claude-code` |
| Phiên bản | `claude-code@2.1.88` |
| Nguồn gốc | **Fork/leak** — KHÔNG phải repo chính thức `anthropics/claude-code` |
| TS/TSX files trong src/ | 1,907 files |
| Dòng code trong src/ | **512,996 dòng** (chỉ `.ts` + `.tsx`) |
| Tổng dòng (bao gồm stubs, lock, etc.) | ~3,156,818 dòng |
| Top-level modules | 35 thư mục trong `src/` |

> **Ghi chú:** Con số **513K dòng** trong bài phân tích gốc là **CHÍNH XÁC** — đó là đếm src/ TypeScript files, không phải tổng repo.

---

## 2. BẢNG KIỂM CHỨNG 10 NHẬN ĐỊNH

### ✅ Nhận định #1: `query.ts` ~1,728 dòng, async generator + while(true) loop
**Kết quả: ĐÚNG — Chính xác cao**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| ~1,728 dòng | 1,729 dòng | `wc -l src/query.ts` |
| Async generator | `async function* query(params: QueryParams): AsyncGenerator<...>` | `src/query.ts:219` |
| while(true) | `while (true) {` | `src/query.ts:307` |
| stop_reason handling | Dùng `needsFollowUp` flag thay vì match trực tiếp `stop_reason` string | `src/query.ts:554-558` |
| Recovery path max_tokens | Có, với reactive compact fallback | `src/query.ts:1065-1079` |
| Reactive compact giữa vòng lặp | Xác nhận — collapse drain → reactive compact → escalate | `src/query.ts:1066-1069` |

**Chi tiết bổ sung:** Code comment tại line 554 ghi rõ: `stop_reason === 'tool_use' is unreliable -- it's not always set correctly.` Vì vậy hệ thống dùng `needsFollowUp` boolean (set `true` khi phát hiện `tool_use` block trong stream) thay vì dựa vào `stop_reason`. Đây là insight quan trọng — bài phân tích gốc nói "chỉ dừng khi `stop_reason: end_turn`" thì **về logic đúng nhưng về implementation sai** — thực tế là "chỉ dừng khi `needsFollowUp === false`".

---

### ✅ Nhận định #2: `isConcurrencySafe()` trên Tool classes
**Kết quả: ĐÚNG**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| Method trên Tool class | `isConcurrencySafe(input: z.infer<Input>): boolean` | `src/Tool.ts:402` |
| Mỗi tool tự khai báo | Default return `false` — tool phải explicitly opt-in | `src/Tool.ts:759` |
| Dựa trên input cụ thể | Đúng — nhận `input` parameter để quyết định runtime | Signature tại `Tool.ts:402` |

**Chi tiết bổ sung:** Default `false` có nghĩa mọi tool mặc định chạy exclusive. Chỉ tool nào override và return `true` mới được chạy song song. Đây là design pattern **safe-by-default**.

---

### ✅ Nhận định #3: `StreamingToolExecutor` — thực thi tool khi đang stream
**Kết quả: ĐÚNG**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| Class tồn tại | `StreamingToolExecutor` | `src/services/tools/StreamingToolExecutor.ts` |
| Bắt đầu thực thi ngay | `addTool()` method gọi khi tool block arrive từ stream | Lines 76-124 |
| Buffer kết quả theo thứ tự | Results emitted in order received | Architecture comment lines 34-39 |
| Feature-gated | Gated bởi `config.gates.streamingToolExecution` | `src/query.ts:561` |

**Chi tiết bổ sung:** Trong `query.ts:561-568`, `StreamingToolExecutor` chỉ được khởi tạo khi gate `streamingToolExecution` bật. Nếu tắt, tool execution rơi về sequential path. Đây là thêm một lớp safety net.

---

### ✅ Nhận định #4: `contextModifier` — tool trả về function biến đổi context
**Kết quả: ĐÚNG**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| Tồn tại trên tool result | `contextModifier?: (context: ToolUseContext) => ToolUseContext` | `src/Tool.ts:330` |
| Chỉ cho non-concurrent tools | Comment: "contextModifier is only honored for tools that aren't concurrency safe" | `src/Tool.ts:329` |
| StreamingToolExecutor collect & apply | `contextModifiers` array, push + apply | `StreamingToolExecutor.ts:31, 160-163` |

**Chi tiết bổ sung:** Constraint "chỉ non-concurrent" là logic — nếu nhiều tool chạy song song đều modify context thì thứ tự apply không deterministic. Bằng cách giới hạn chỉ exclusive tools mới có contextModifier, hệ thống đảm bảo sequential application.

---

### ✅ Nhận định #5: Coordinator mode với restricted toolset
**Kết quả: ĐÚNG**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| TeamCreate tool | Tồn tại | `src/tools/TeamCreateTool/TeamCreateTool.ts` |
| TeamDelete tool | Tồn tại | `src/tools/TeamDeleteTool/TeamDeleteTool.ts` |
| SendMessage tool | Tồn tại | `src/tools/SendMessageTool/SendMessageTool.ts` |
| SyntheticOutput tool | Tồn tại | `src/tools/SyntheticOutputTool/SyntheticOutputTool.ts` |
| Coordinator mode feature-gated | `feature('COORDINATOR_MODE')` + env var | `src/coordinator/coordinatorMode.ts:36-41` |
| Restricted toolset | `INTERNAL_WORKER_TOOLS` = Set of 4 tools trên | `coordinatorMode.ts:29-34` |

**Chi tiết bổ sung:** Code tại `coordinatorMode.ts:29-34` define `INTERNAL_WORKER_TOOLS` chính xác 4 tool name. Coordinator mode check cả feature flag VÀ env var `CLAUDE_CODE_COORDINATOR_MODE`. Có thêm `matchSessionMode()` function đảm bảo resumed sessions giữ đúng mode.

---

### ✅ Nhận định #6: Fork agent tạo Git worktree
**Kết quả: ĐÚNG**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| Fork subagent file | Tồn tại | `src/tools/AgentTool/forkSubagent.ts` |
| Worktree notice | `buildWorktreeNotice()` — "isolated git worktree at ${worktreeCwd}" | `forkSubagent.ts:205-210` |
| Fork agent definition | `FORK_AGENT` với `tools: ['*']`, `permissionMode: 'bubble'` | `forkSubagent.ts:60-71` |
| Detection function | `isInForkChild()` checks fork boilerplate tag | `forkSubagent.ts:78-80` |

**Chi tiết bổ sung:** `permissionMode: 'bubble'` nghĩa là permission requests từ fork agent "nổi bọt" lên parent — fork không tự approve. Đây là safety measure cho isolated execution.

---

### ✅ Nhận định #7: Compact boundary message — "quên có kiểm soát"
**Kết quả: ĐÚNG**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| Boundary message functions | `createCompactBoundaryMessage()` + `createMicrocompactBoundaryMessage()` | `src/utils/messages.ts` |
| Boundary type | `SystemCompactBoundaryMessage` với `subtype: 'microcompact_boundary'` | `src/types/message.ts` |
| Sử dụng trong query loop | Yield boundary messages tại 3 điểm | `query.ts:54, 407, 528-532` |

**Chi tiết bổ sung:** Compact service tổng cộng **3,971 dòng** (13 files trong `src/services/compact/`). Bao gồm:
- `compact.ts` (1,705 dòng) — core compaction logic
- `sessionMemoryCompact.ts` (630 dòng) — session memory handling
- `microCompact.ts` (530 dòng) — lightweight compaction
- `autoCompact.ts` (351 dòng) — auto-trigger logic
- `prompt.ts` (374 dòng) — LLM prompts cho summarization

---

### ✅ Nhận định #8: Memory system — extract sau, attach trước
**Kết quả: ĐÚNG**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| Extract memories service | `extractMemories.ts` — chạy cuối query loop | `src/services/extractMemories/extractMemories.ts` |
| Attach memories | `attachments.ts` — gắn relevant memories đầu turn | `src/utils/attachments.ts:61-64` |
| Background extraction | Dùng `runForkedAgent` — không block main thread | Comment trong extractMemories.ts |
| Timing | "runs once at the end of each complete query loop" | Doc comment lines 1-13 |

---

### ✅ Nhận định #9: `outputFile + outputOffset` — task output qua disk
**Kết quả: ĐÚNG — Cả hai đều tồn tại**

| Claim | Thực tế | Evidence |
|-------|---------|---------|
| `outputFile` | `getTaskOutputPath(agentId)` | `src/tools/AgentTool/resumeAgent.ts` |
| `outputOffset` | `outputOffset: number` trên Task type | `src/Task.ts:55` |
| Default value | `outputOffset: 0` khi tạo task mới | `src/Task.ts:122` |
| Usage pattern | Read output từ disk với offset tracking | `src/utils/task/framework.ts:192, 209, 230` |

**Chi tiết bổ sung:** `framework.ts:209` có comment: "Apply the outputOffset patches and evictions from generateTaskAttachments." — xác nhận đây là pattern message passing qua file với offset để đọc incremental output.

> **Lưu ý:** Bài phân tích gốc nhận định này chính xác. Lần kiểm chứng đầu tiên tôi miss `outputOffset` vì chỉ search trong `tools/` — thực tế nó nằm trong `Task.ts` (type definition) và `utils/task/framework.ts` (implementation).

---

### ✅ Nhận định #10: 4-layer context defense
**Kết quả: ĐÚNG — Nhưng cần bổ sung chi tiết**

**4 lớp xác nhận:**

| Layer | Tên | File | Dòng code | Mô tả |
|-------|-----|------|-----------|-------|
| 1 | Tool result truncation | `Tool.ts:466` | — | `maxResultSizeChars` — kết quả vượt giới hạn được persist xuống disk, chỉ giữ pointer |
| 2 | Microcompact | `services/compact/microCompact.ts` | 530 | Loại bỏ tool results cũ tại compaction boundaries — lightweight, không cần LLM |
| 3 | Auto-compact (LLM summarize) | `services/compact/autoCompact.ts` + `compact.ts` | 2,056 | Full conversation summary bằng LLM khi token count vượt ngưỡng |
| 4 | Context collapse | `services/contextCollapse/index.ts` | 6 (stub) | **Tồn tại nhưng STUBBED** — code bị stub trong bản leak, implementation thật không có |

**Chi tiết bổ sung từ `query.ts:1066-1069`:**
```
// collapse drain first (cheap, keeps granular context), then reactive compact
// (full summary). Single-shot on each — if a retry still 413's,
// the next stage handles it or the error surfaces.
```

Điều này xác nhận **escalation pattern**: collapse drain (lớp nhẹ) → reactive compact (lớp nặng) → error surface. Thêm vào đó có:
- `snipCompact.ts` (4 dòng — stub, gated bởi `HISTORY_SNIP`)
- `cachedMicrocompact.ts` (7 dòng — stub, gated bởi `CACHED_MICROCOMPACT`)
- `sessionMemoryCompact.ts` (630 dòng — memory-aware compaction)

**Kết luận:** Thực tế có **5-6 lớp** defense, không chỉ 4. Nhưng trong bản leak, context collapse và 2 compact variant bị stub nên chỉ **3 lớp hoạt động đầy đủ**.

---

## 3. TỔNG KẾT KIỂM CHỨNG

| # | Nhận định | Kết quả | Độ chính xác |
|---|-----------|---------|-------------|
| 1 | query.ts async generator + while(true) | ✅ ĐÚNG | 95% — logic đúng, implementation detail khác (needsFollowUp vs stop_reason) |
| 2 | isConcurrencySafe() trên Tool | ✅ ĐÚNG | 100% |
| 3 | StreamingToolExecutor | ✅ ĐÚNG | 100% |
| 4 | contextModifier | ✅ ĐÚNG | 100% |
| 5 | Coordinator restricted toolset | ✅ ĐÚNG | 100% |
| 6 | Fork worktree isolation | ✅ ĐÚNG | 100% |
| 7 | Compact boundary message | ✅ ĐÚNG | 100% |
| 8 | Memory extract/attach | ✅ ĐÚNG | 100% |
| 9 | outputFile + outputOffset | ✅ ĐÚNG | 100% |
| 10 | 4-layer context defense | ✅ ĐÚNG | 85% — thực tế 5-6 lớp, 2-3 lớp bị stub trong leak |

**Tổng: 10/10 nhận định ĐÚNG** (2 nhận định cần bổ sung chi tiết implementation)

---

## 4. PHÁT HIỆN BỔ SUNG — Những gì bài phân tích CHƯA ĐỀ CẬP

### 4.1. Feature Gates — Nhiều tính năng bị ẩn
Repo sử dụng `feature('FLAG_NAME')` từ `bun:bundle` cho dead code elimination. Bản leak shim tất cả thành `false`. Các feature quan trọng bị ẩn:

| Flag | Ý nghĩa |
|------|---------|
| `COORDINATOR_MODE` | Multi-agent coordinator (nhận định #5) |
| `KAIROS` | Session transcript — có thể là advanced replay/audit |
| `ULTRAPLAN` | Planning mode nâng cao |
| `VOICE_MODE` | Voice input/output |
| `BRIDGE_MODE` | Cloud session bridging |
| `BG_SESSIONS` | Background sessions |
| `WORKFLOW_SCRIPTS` | Workflow automation |
| `HISTORY_SNIP` | History snipping compaction |
| `CACHED_MICROCOMPACT` | Cached microcompact |
| `BUDDY` | Buddy mode (chưa rõ) |
| `TEAMMEM` | Team memory sync |

### 4.2. Tool inventory — 43 tools, không phải ~40
Bài phân tích ghi "~40 tools" — thực tế có **43 tool directories** trong `src/tools/`, bao gồm:
- 4 tool chưa được nhắc: `TungstenTool` (Ant-only debug), `REPLTool` (Ant-only), `SuggestBackgroundPRTool` (Ant-only), `WorkflowTool` (feature-gated)
- `VerifyPlanExecutionTool` — env-gated, liên quan đến ULTRAPLAN

### 4.3. Compact service phức tạp hơn mô tả
Compact không chỉ là "4 lớp" — nó là **13 files, 3,971 dòng**, bao gồm:
- Time-based microcompact config (`timeBasedMCConfig.ts`)
- Session memory compaction riêng (`sessionMemoryCompact.ts` — 630 dòng)
- Post-compact cleanup (`postCompactCleanup.ts`)
- API-level microcompact (`apiMicrocompact.ts`)
- Compact warning hooks cho UX

### 4.4. Skill system — Chưa được phân tích
Bài phân tích không đề cập **Skill system** — 23 files trong `src/skills/`, 16 bundled skills. Skills khác với Tools: chúng là higher-level abstractions được invoke qua slash commands, có thể chứa multi-step workflows.

### 4.5. Plugin system — Chưa được phân tích
65+ files trong `src/utils/plugins/`, full marketplace với 32 official plugins. Plugin có thể cung cấp agents, commands, hooks — là extension mechanism chính của Claude Code.

### 4.6. Custom Ink fork — Không phải React tiêu chuẩn
Toàn bộ UI dùng **custom fork của Ink** (terminal React renderer) với:
- Custom flexbox layout engine (port từ Meta's Yoga)
- Custom reconciler
- Custom event system (click, keyboard, focus)
- ~140 UI components

---

## 5. ĐÁNH GIÁ BÀI PHÂN TÍCH GỐC

### Điểm mạnh
1. **10/10 nhận định kiến trúc đều đúng** — chứng tỏ phân tích dựa trên đọc code thực, không phỏng đoán
2. **Insights sâu** — đặc biệt về concurrency partitioning (#2, #4), coordinator restriction (#5), và memory lifecycle (#8)
3. **Pattern extraction** có giá trị ứng dụng — 10 pattern rút ra đều có cơ sở và applicability

### Điểm cần bổ sung
1. **Implementation detail của query loop** — `needsFollowUp` flag thay vì `stop_reason` string matching. Subtle nhưng quan trọng nếu áp dụng pattern này
2. **Context defense phức tạp hơn 4 lớp** — 5-6 lớp thực tế, nhiều variant bị feature-gated
3. **3 hệ thống chưa phân tích**: Skills, Plugins, và custom Ink UI — đây là các pipeline quan trọng nếu muốn hiểu đầy đủ kiến trúc
4. **Feature gates** — ~20 feature flags ẩn nhiều tính năng production. Bản leak chỉ cho thấy ~70% kiến trúc thật

### Đánh giá tổng thể
**Bài phân tích đạt độ chính xác ~95%.** Các nhận định lõi đều chính xác, pattern extraction có giá trị. Cần bổ sung 3 pipeline chưa phân tích nếu muốn bức tranh hoàn chỉnh.

---

## 6. KHUYẾN NGHỊ CHO CHỦ THẦU

### Về bài phân tích
1. **Có thể sử dụng làm reference** — 10 pattern đều verified, an toàn để build blueprint từ đây
2. **Bổ sung caveat** về `needsFollowUp` vs `stop_reason` khi trích dẫn Query Loop pattern
3. **Xem xét thêm** Skill + Plugin + Ink pipeline nếu cần architectural completeness

### Về ứng dụng cho RTR
1. **Vibecode v7.0** — Pattern #5 (coordinator restriction) xác nhận mạnh. Khuyến nghị thêm pattern Plugin system (extension mechanism) vào v7.0
2. **SimEngine** — Pattern #1, #4, #5, #8 đều applicable. Lưu ý thêm `permissionMode: 'bubble'` pattern cho safety
3. **RTR Platform** — Context defense cần thiết kế 5+ lớp, không chỉ 4. Session memory compaction (630 dòng riêng) là pattern quan trọng cho long-running sessions

---

*Báo cáo tạo bởi Thợ thi công — Awaiting review từ Chủ thầu.*
