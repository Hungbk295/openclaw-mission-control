# OpenClaw Mission Control — Flow Tutorial Breakdown

> Hướng dẫn từng flow như một video tutorial.
> Mỗi phần giải thích theo phong cách: "Bạn đang đứng ở đây, bạn bấm nút này, chuyện gì sẽ xảy ra bên trong?"

---

## Mục lục

1. [Flow 1: Authentication — "Ai đang gõ cửa?"](#flow-1-authentication--ai-đang-gõ-cửa)
2. [Flow 2: Board & Task — "Bàn cờ và quân cờ"](#flow-2-board--task--bàn-cờ-và-quân-cờ)
3. [Flow 3: Agent Lifecycle — "Tuyển nhân viên robot"](#flow-3-agent-lifecycle--tuyển-nhân-viên-robot)
4. [Flow 4: Approval Workflow — "Xin phê duyệt sếp"](#flow-4-approval-workflow--xin-phê-duyệt-sếp)
5. [Flow 5: Gateway — "Cổng kết nối thế giới bên ngoài"](#flow-5-gateway--cổng-kết-nối-thế-giới-bên-ngoài)
6. [Flow 6: Webhook — "Chuông báo tự động"](#flow-6-webhook--chuông-báo-tự-động)
7. [Flow 7: Onboarding — "Hướng dẫn người mới"](#flow-7-onboarding--hướng-dẫn-người-mới)
8. [Flow 8: Activity Feed — "Nhật ký mọi chuyện"](#flow-8-activity-feed--nhật-ký-mọi-chuyện)

---

## Flow 1: Authentication — "Ai đang gõ cửa?"

### Liên tưởng

Hãy tưởng tượng Mission Control là một toà nhà văn phòng. Trước khi vào được, bạn cần qua cổng bảo vệ. Hệ thống có **2 kiểu bảo vệ**:

- **Local Mode** (Bảo vệ dùng mật mã): Bạn nói đúng câu mật mã → vào được. Ai cũng dùng chung 1 mật mã.
- **Clerk Mode** (Bảo vệ dùng thẻ nhân viên): Mỗi người có thẻ riêng, quẹt thẻ mới vào được.

Còn **Agent** (robot) thì có cửa riêng, dùng **badge đặc biệt** (X-Agent-Token).

### Bước 1: Ứng dụng khởi động — "Mở cửa chính"

Khi user mở trình duyệt truy cập app, file đầu tiên chạy là `AuthProvider`:

```
frontend/src/components/providers/AuthProvider.tsx
```

AuthProvider đọc biến môi trường `NEXT_PUBLIC_AUTH_MODE` để quyết định kiểu bảo vệ nào:

```
Nếu AUTH_MODE = "clerk" VÀ có CLERK_KEY hợp lệ (pk_test_... hoặc pk_live_..., body >= 16 ký tự)
  → Bọc app trong ClerkProvider (hệ thống thẻ nhân viên)
  → Xoá token local cũ nếu có (tránh xung đột)

Nếu AUTH_MODE = "local" HOẶC không set gì
  → Dùng LocalAuthGate (hệ thống mật mã)
```

### Bước 2a: Local Auth — "Nói mật mã đi"

**Người dùng thấy gì:** Màn hình login với 1 ô nhập token.

**Bên trong xảy ra gì:**

```
1. LocalAuthGate mount → check sessionStorage có key "mc_local_auth_token" không
   ├── Có token → render app ngay (bạn đã vào rồi)
   └── Không có → hiện form LocalAuthLogin

2. User nhập token (tối thiểu 50 ký tự), bấm "Sign In"
   → GET /api/v1/users/me (kèm header Authorization: Bearer <token>)

3. Backend nhận request:
   → Tách bearer token từ header
   → So sánh với LOCAL_AUTH_TOKEN trong env (dùng hmac.compare_digest — chống timing attack)
   → LOCAL_AUTH_TOKEN phải >= 50 ký tự, không được là placeholder ("change-me", "changeme"...)
   ├── Khớp → tạo/lấy user "local-auth-user", trả user profile
   └── Không khớp → trả 401 "Invalid token"

4. Frontend nhận 200:
   → Lưu token vào sessionStorage key "mc_local_auth_token" (mất khi đóng tab)
   → window.location.reload() → app hiện ra
```

**Điểm thú vị:**
- `sessionStorage` (không phải localStorage): đóng tab = logout tự động
- Storage key: `mc_local_auth_token`
- `hmac.compare_digest`: so sánh từng byte cùng tốc độ, hacker không thể đoán từng ký tự
- Chỉ có 1 user duy nhất cho local mode: ID cố định `"local-auth-user"`
- Token tối thiểu 50 ký tự, không chấp nhận placeholder values

```
                                              ┌─────────────────┐
 User nhập token ──► GET /users/me ──────────►│  compare_digest  │
                                              │  token == ENV?   │
                                              └────────┬────────┘
                                                  ✅ │  ❌
                                        ┌─────────┘    └──────────┐
                                        ▼                         ▼
                                  sessionStorage            401 "Invalid"
                                   lưu token
                                        │
                                        ▼
                                 window.location.reload()
```

### Bước 2b: Clerk Auth — "Quẹt thẻ nhân viên"

**Người dùng thấy gì:** Trang đăng nhập của Clerk (email, Google, GitHub...).

```
1. ClerkProvider bọc toàn bộ app
2. Chưa login → redirect tới /sign-in
3. /sign-in render <SignIn /> component của Clerk
4. User đăng nhập qua Clerk (email, Google, GitHub...) → nhận JWT cookie
5. Mỗi API call:
   → mutator.ts gọi Clerk.session.getToken() → lấy JWT mới
   → Gửi kèm header Authorization: Bearer <jwt>
6. Backend verify JWT bằng Clerk SDK:
   → authenticate_request() verify signature + clock skew (±10s)
   → Extract user profile từ JWT claims
   → Sync user vào DB local (email, tên, ảnh)
   → Nếu thiếu info → gọi Clerk API lấy full profile (timeout 5s, fail = log only)
   → Đảm bảo user có organization membership
   → Trả AuthContext cho route handler
7. Bootstrap endpoint: POST /api/v1/auth/bootstrap
   → Frontend gọi sau khi có token để lấy user profile đầy đủ
   → Trả về UserRead (id, email, name, preferences, admin status)
```

### Bước 3: Token đi theo mọi request — "Thẻ ra vào luôn trên cổ"

File `mutator.ts` là "người vận chuyển thẻ". Mọi API call từ frontend đều đi qua đây:

```
frontend/src/api/mutator.ts
```

Thứ tự ưu tiên:
```
1. Có local token trong sessionStorage? → Dùng nó
2. Không có? → Lấy JWT từ Clerk session
3. Đều không có? → Gửi request không có auth (sẽ bị 401)
```

### Bước 4: Agent Auth — "Cửa dành cho robot"

Agent không login qua giao diện. Chúng dùng **X-Agent-Token** header (hoặc fallback Bearer token):

```
Agent gửi request:
  Header: X-Agent-Token: <plain-token>  (ưu tiên)
  HOẶC:  Authorization: Bearer <plain-token>  (fallback)

Backend nhận:
  1. Kiểm tra X-Agent-Token TRƯỚC (ưu tiên hơn user auth)
  2. Duyệt tất cả agent có agent_token_hash trong DB
  3. Verify token bằng PBKDF2-HMAC-SHA256 (200,000 iterations, constant-time comparison)
  4. Agent.is_active == true? → OK
  5. Cập nhật last_seen_at (throttle 30s), status = "online"
  6. Trả AuthContext(actor_type="agent", agent=agent)
```

**Bảo mật token:**
- Token gốc chỉ hiện **1 lần duy nhất** khi tạo agent
- DB lưu PBKDF2 hash: `pbkdf2_sha256$200000${salt}${digest}` (không thể đảo ngược)
- Constant-time comparison chống timing attack
- Giống cách GitHub Personal Access Token hoạt động

### Tóm tắt Authentication

| Ai? | Dùng gì? | Gửi qua? | Backend check? |
|-----|----------|-----------|----------------|
| User (local) | Bearer token (>=50 chars) | Authorization header | hmac.compare_digest vs ENV |
| User (clerk) | JWT | Authorization header | Clerk SDK verify + user sync |
| Agent | API token | X-Agent-Token (hoặc Bearer) | PBKDF2 hash verify (200K iter) |

---

## Flow 2: Board & Task — "Bàn cờ và quân cờ"

### Liên tưởng

Board giống như một **bảng Kanban treo tường** trong phòng họp:
- **Board** = tấm bảng (có nhiều cột)
- **Columns** = các cột trên bảng ("To Do", "Doing", "Done")
- **Task** = tờ sticky note dán trên bảng
- **Position** = vị trí sticky note trong cột (trên/dưới)

### Bước 1: Tạo Board — "Treo bảng mới lên tường"

**User thấy gì:** Bấm "New Board" → form hiện ra (tên, mô tả, loại).

**Bên trong:**

```
POST /api/v1/boards
Body: { name: "Sprint 42", description: "...", board_type: "kanban" }

Backend:
1. Verify user là org member
2. Tạo Board:
   - slug = slugify("Sprint 42") → "sprint-42"
   - column_config = DEFAULT_COLUMNS (5 cột mặc định)
   - lead_user_id = user hiện tại
3. Tự động thêm creator làm BoardMember với role "lead"
4. Ghi activity: "board.created"
5. Trả về BoardRead
```

**5 cột mặc định:**
```
┌──────────┬──────────┬────────────┬──────────┬──────────┐
│ Backlog  │  To Do   │In Progress │  Review  │   Done   │
│  (xám)   │  (xanh)  │  (vàng)    │  (tím)   │ (xanh lá)│
│          │          │            │          │ is_done  │
└──────────┴──────────┴────────────┴──────────┴──────────┘
```

Mỗi cột có: `name` (key), `label` (hiển thị), `color`, `wip_limit` (giới hạn), `is_done` (đánh dấu cột hoàn thành).

### Bước 2: Xem Board — "Nhìn vào bảng"

```
User vào /boards/{board_id}

Frontend gửi 2 request song song:
  1. GET /api/v1/boards/{board_id} → Thông tin board + column_config
  2. GET /api/v1/tasks?board_id={board_id} → Tất cả task

Render:
  Với mỗi column trong column_config:
    → Lọc tasks có status === column.name
    → Render KanbanColumn chứa các TaskCard

Cấu trúc component:
  BoardDetailPage
  ├── BoardHeader (tên, mô tả, settings)
  ├── BoardToolbar (filters, search)
  └── KanbanBoard
      ├── KanbanColumn "Backlog" → [TaskCard, TaskCard]
      ├── KanbanColumn "To Do" → [TaskCard, TaskCard, TaskCard]
      ├── KanbanColumn "In Progress" → [TaskCard]
      ├── KanbanColumn "Review" → [TaskCard]
      └── KanbanColumn "Done" → [TaskCard, TaskCard]
```

### Bước 3: Tạo Task — "Dán sticky note mới"

**2 cách tạo:**

**Cách nhanh** (inline): Bấm "+" trong cột → nhập title → Enter
```
POST /api/v1/tasks { board_id, title: "Fix login bug", status: "todo" }
```

**Cách đầy đủ** (dialog): Mở form → điền title, description, priority, assignee, tags, due date
```
POST /api/v1/tasks { board_id, title, description, priority, assignee_id, tag_ids, due_date }
```

**Backend xử lý:**
```
1. Verify board write access
2. Tính position = max_position hiện tại + 1000
   (Tại sao +1000? Để sau này chèn giữa 2 task dễ dàng.
    VD: task A ở 1000, task B ở 2000 → chèn C ở 1500)
3. Tạo Task record
4. Gắn tags qua bảng trung gian TaskTag
5. Ghi activity: "task.created"
6. Gửi webhook (nếu board có cấu hình)
7. Trả về TaskRead
```

### Bước 4: Kéo thả Task — "Di chuyển sticky note"

**User kéo task từ "To Do" sang "In Progress":**

```
Frontend:
  1. Drag start → ghi nhớ vị trí cũ
  2. Drop → tính position mới dựa trên vị trí thả
  3. Optimistic update: card di chuyển NGAY trước khi API trả về
  4. POST /api/v1/tasks/batch-move {
       board_id,
       moves: [{ task_id, status: "in_progress", position: 1500 }]
     }

Backend:
  1. Cập nhật task.status, task.position
  2. Nếu chuyển từ "todo" → bất kỳ cột khác:
     → Đặt task.started_at = now (lần đầu bắt đầu)
  3. Nếu chuyển vào cột "done":
     → Đặt task.completed_at = now
  4. Nếu chuyển từ "done" ra:
     → Xoá task.completed_at (task được mở lại)
  5. KIỂM TRA DEPENDENCIES:
     → Task có bị block bởi task khác chưa hoàn thành?
     → Nếu có → 409 "Task blocked by N dependencies"
  6. Ghi activity: "task.status_changed" (kèm old/new status)
  7. Gửi webhook

Nếu API lỗi:
  → Frontend revert về vị trí cũ (undo optimistic update)
```

```
             Kéo task "Fix bug"
                    │
    ┌───────────────┼───────────────┐
    │   To Do       │  In Progress  │
    │               │               │
    │  ┌─────────┐  │               │
    │  │ Fix bug │──┼──►  ┌─────────┐
    │  └─────────┘  │     │ Fix bug │
    │               │     └─────────┘
    │  ┌─────────┐  │               │
    │  │ Add API │  │               │
    │  └─────────┘  │               │
    └───────────────┴───────────────┘

    Đồng thời: PATCH task.status = "in_progress"
               task.started_at = now()
               activity "task.status_changed" logged
```

### Bước 5: Task Dependencies — "Task này phải xong trước task kia"

Giống như xây nhà: phải đổ móng xong mới xây tường.

```
User mở task detail → mục Dependencies → "Add Dependency"
  → Chọn task prerequisite
  → POST /api/v1/tasks/{task_id}/dependencies
    Body: { depends_on_id: prerequisite_id, dependency_type: "blocks" }

Backend kiểm tra VÒNG TRÒN (circular dependency):
  → Dùng BFS duyệt từ prerequisite:
    "Nếu đi theo chuỗi dependency từ prerequisite,
     có quay lại task hiện tại không?"
  → Nếu có vòng tròn → 409 "Would create circular dependency"
  → Không có → tạo TaskDependency record

Hiệu ứng:
  Khi cố kéo task bị block sang cột active:
  → Backend check: prerequisite đã "done" chưa?
  → Chưa → 409 "Task blocked by N dependencies"
  → Frontend hiện cảnh báo
```

```
  Task A ──blocks──► Task B ──blocks──► Task C
  (phải xong A)     (mới làm B)       (mới làm C)

  Nếu thêm C blocks A → VÒNG TRÒN → từ chối!
```

### Bước 6: Board Snapshots — "Chụp ảnh bảng mỗi ngày"

```
Định kỳ hoặc khi admin trigger:
  → create_board_snapshot()
  → Đếm tất cả tasks theo status, priority
  → Lưu BoardSnapshot: {
      total_tasks: 42,
      status_breakdown: { todo: 10, in_progress: 8, done: 24 },
      priority_breakdown: { high: 5, medium: 30, low: 7 }
    }
  → Dùng cho: burndown chart, velocity tracking, báo cáo lịch sử
```

---

## Flow 3: Agent Lifecycle — "Tuyển nhân viên robot"

### Liên tưởng

Tưởng tượng bạn đang quản lý một đội ngũ robot trong nhà máy:
- **Tạo agent** = tuyển robot mới, cấp thẻ nhân viên
- **Provision** = đăng ký robot vào hệ thống nhà máy + cấp tài liệu hướng dẫn
- **Heartbeat** = robot báo "tôi vẫn sống" mỗi 10 phút
- **Board Lead** = quản đốc robot, mỗi board có 1 lead
- **Worker** = robot công nhân, nhận task từ lead hoặc tự nhận

### Bước 1: Đăng ký Agent — "Tuyển robot mới"

```
Admin vào /agents/new → Form: tên, board, role, emoji, communication style, heartbeat interval

POST /api/v1/agents
Body: { name: "CodeBot", board_id: "...", identity_profile: { role: "...", emoji: ":gear:" } }

Backend (AgentLifecycleService.create_agent):
  1. Validate: board + gateway phải tồn tại
  2. Enforce spawn limits (max_agents per board, excluding lead)
  3. Ensure unique name within board + gateway workspace
  4. Tạo token: mint_agent_token() → 43-char URL-safe random token
  5. Hash token: PBKDF2-HMAC-SHA256 (200K iterations, 16-byte random salt)
  6. Resolve session key (lead → board-based key, worker → UUID-based key)
  7. Tạo Agent record (status: "provisioning")
  8. Trigger provisioning: AgentLifecycleOrchestrator.run_lifecycle()
  9. Trả về: { ...agent_data, agent_token: "<plain-token>" }
                                          ⬆ CHỈ HIỆN 1 LẦN DUY NHẤT!

Frontend:
  → Hiện token display: "Copy token này ngay. Nó sẽ KHÔNG hiện lại."
  → Agent list auto-refetch mỗi 15 giây để thấy status thay đổi
```

```
  ┌──── Lần đầu tạo ──────────────────────────────────────┐
  │                                                       │
  │  Raw Token: "sk_agent_a8f3k2..."                      │
  │       │                                               │
  │       ├──► PBKDF2(200K iter, 16-byte salt) ──► DB hash│
  │       │    Format: pbkdf2_sha256$200000$salt$digest    │
  │       │                                               │
  │       ├──► Embed vào TOOLS.md (trên gateway)          │
  │       │                                               │
  │       └──► Hiện cho admin (1 lần duy nhất)            │
  │                                                       │
  └───────────────────────────────────────────────────────┘

  Sau đó: Token gốc KHÔNG CÒN ở đâu trong hệ thống (trừ gateway workspace).
          Mất = phải regenerate token mới.
```

### Bước 2: Provisioning — "Chuẩn bị chỗ làm việc cho robot"

Đây là bước quan trọng nhất. Giống như khi nhân viên mới vào công ty: cần chuẩn bị
bàn làm việc, tài liệu onboarding, badge, và danh sách nhiệm vụ.

```
OpenClawGatewayProvisioner.apply_agent_lifecycle() thực hiện 3 giai đoạn:

GIAI ĐOẠN 1: Đăng ký agent trên Gateway
  → RPC gọi agents.create → agents.update trên gateway
  → Tạo workspace directory trên gateway filesystem
  → Đăng ký agent trong gateway runtime

GIAI ĐOẠN 2: Đồng bộ template files (Jinja2 rendering)
  Hệ thống render và upload các file hướng dẫn cho agent:

  ┌─ Chung cho mọi agent ────────────────────────────────┐
  │ IDENTITY.md  — "Bạn là ai?" (role, communication style)│
  │ SOUL.md      — "Bạn nghĩ gì?" (system prompt sâu)   │
  │ HEARTBEAT.md — "Bạn phải báo cáo thế nào?"          │
  │ BOOTSTRAP.md — "Lần đầu bạn làm gì?"                │
  │ TOOLS.md     — "Bạn có gì?" (auth token, tool defs)  │
  │ AGENTS.md    — "Đồng nghiệp của bạn là ai?"         │
  │ ROUTING.md   — "Ai gửi tin nhắn cho ai?"             │
  │ LEARNINGS.md — "Bạn đã học được gì?"                 │
  └───────────────────────────────────────────────────────┘

  ┌─ Thêm cho Board Lead ────────────────────────────────┐
  │ USER.md      — "Sếp của bạn muốn gì?"               │
  │ ROLE.md      — "Vai trò quản đốc"                    │
  │ WORKFLOW.md  — "Quy trình làm việc"                  │
  └───────────────────────────────────────────────────────┘

  Lưu ý: Một số file "editable" (USER.md, MEMORY.md) được BẢO VỆ
  khi update — không bị ghi đè trừ khi overwrite=true.

GIAI ĐOẠN 3: Đánh thức agent (wake=true)
  → ensure_session() → tạo OpenClaw session
  → send_message() → gửi tin nhắn đánh thức
  → Tin nhắn: "Đọc BOOTSTRAP.md, rồi AGENTS.md, rồi bắt đầu heartbeat"
  → Đặt checkin_deadline_at = now + 5 phút (deadline check-in)
```

### Bước 3: Agent bắt đầu làm việc — "Robot vào ca"

```
Agent được đánh thức bởi gateway:
  1. Đọc BOOTSTRAP.md → hiểu mình cần làm gì lần đầu
  2. Đọc AGENTS.md → biết đồng nghiệp robot khác
  3. Gửi heartbeat đầu tiên:
     → POST /api/v1/agents/{agent_id}/heartbeat
     → Backend: last_seen_at = now, status: "provisioning" → "online"
     → Reset wake_attempts = 0, checkin_deadline_at = null

  4. Bắt đầu vòng lặp làm việc:
     → GET /api/v1/agent/tasks → xem task cần làm
     → Làm task, gửi comments cập nhật tiến độ
     → Heartbeat mỗi 10 phút (configurable per-agent)
```

### Bước 4: Agent nhận và làm task — "Robot bắt tay vào việc"

```
Agent chọn task phù hợp:
  → POST /api/v1/agent/tasks/{task_id}/claim
  → Gán task.assignee_agent_id = agent.id
  → task.status = "in_progress"
  → Activity: "task.claimed"

Trong khi làm, agent gửi cập nhật:
  → POST /api/v1/agent/tasks/{task_id}/comments
    Body: { content: "Đang xử lý bước 3/5..." }

Khi hoàn thành:
  → POST /api/v1/agent/tasks/{task_id}/complete
    Body: { result: "Đã fix 3 bugs", notes: "Chi tiết..." }
  → task.status = "done", task.completed_at = now
  → Activity: "task.completed", Webhook dispatched

Hoặc gửi review:
  → POST /api/v1/agent/tasks/{task_id}/submit-for-review
  → task.status = "review"
  → Human reviewer xem xét

Board Lead agent có thêm quyền:
  → Tạo/xoá worker agents trên board mình
  → Broadcast tin nhắn cho tất cả agents
  → Nudge agent khác: POST /api/v1/agent/nudge
  → Hỏi user: tạo approval request
```

### Bước 5: Agent gặp sự cố — "Robot bị kẹt"

```
Agent không gửi heartbeat (mất tích):
  → Sau ~10 phút không heartbeat → computed status = "offline"
    (with_computed_status() tính lại dựa trên last_seen_at)

Wake Escalation — "Cố đánh thức robot":
  1. Lần đầu hết deadline (5 phút sau wake):
     → Lifecycle reconciliation job chạy
     → Re-provision agent (gửi lại template files)
     → Gửi wakeup message lần nữa
     → wake_attempts += 1
     → Đặt checkin_deadline_at mới

  2. Lặp lại tối đa 5 lần (MAX_WAKE_ATTEMPTS_WITHOUT_CHECKIN)

  3. Sau 5 lần vẫn không check-in:
     → status → "offline" vĩnh viễn
     → Error logged
     → Admin cần can thiệp

Khi agent heartbeat lại thành công:
  → wake_attempts = 0
  → checkin_deadline_at = null
  → last_provision_error = null
  → status = "online"
  → Vòng lặp làm việc tiếp tục

Khi agent bị xoá:
  → DELETE /api/v1/agents/{id}
  → Tất cả in-progress tasks → unassigned, status → inbox
  → Deprovision khỏi gateway
  → Thông báo gateway-main agent cleanup
```

### Bước 6: Vòng đời Agent — "Sơ đồ trạng thái"

```
                    ┌──────────────┐
                    │   Created    │
                    └──────┬───────┘
                           │ (provision triggered)
                           ▼
                    ┌──────────────┐
             ┌────►│ provisioning │ ← gửi templates + wakeup message
             │     └──────┬───────┘
             │            │ (first heartbeat received)
             │            ▼
             │     ┌──────────────┐     heartbeat mỗi 10 phút
             │     │    online    │◄────────────────────────┐
             │     └──────┬───────┘                        │
             │            │                                │
             │    (no heartbeat ~10min)                     │
             │            ▼                                │
             │     ┌──────────────┐    (heartbeat lại)     │
             ├────►│   offline    │────────────────────────►┘
             │     └──────┬───────┘
             │            │
             │    (re-provision & wake)
             │     tối đa 5 lần
             │            │
             │     ┌──────┴───────┐
             │     │  Vẫn không   │
             │     │  check-in?   │
             │     └──────┬───────┘
             │            │ → offline vĩnh viễn
             │            │   (cần admin can thiệp)
             │
             │  Các trạng thái đặc biệt:
             │     ┌──────────────┐
             ├────►│   updating   │ ← user/lead trigger update
             │     └──────────────┘
             │     ┌──────────────┐
             ├────►│   deleting   │ ← xoá agent
             │     └──────────────┘
             │     ┌──────────────┐
             └────►│    paused    │ ← board memory command /pause
                   └──────────────┘

  Computed vs Explicit status:
    DB lưu explicit status, nhưng with_computed_status() tính lại:
    - last_seen_at == null && status != updating/deleting → "provisioning"
    - last_seen_at quá cũ (>10min) → "offline"
    - Còn lại → dùng explicit status
```

### Agent Types & Roles — "Các loại robot"

| Concept | Vai trò | Liên tưởng |
|---------|---------|------------|
| **Board Lead** | 1 per board, quản lý workers, tạo/xoá agent, broadcast | Quản đốc nhà máy |
| **Worker** | Nhận task, làm việc, báo cáo | Công nhân dây chuyền |
| **Gateway-Main** | Agent không thuộc board nào, quản lý cấp gateway | Giám đốc nhà máy |

**Identity Profile** — mỗi agent có "tính cách":
```
{
  role: "Code reviewer",           ← vai trò
  communication_style: "concise",  ← phong cách giao tiếp
  emoji: ":gear:"                  ← biểu tượng đại diện
}
```
Render vào IDENTITY.md để agent biết mình cần hành xử thế nào.

---

## Flow 4: Approval Workflow — "Xin phê duyệt sếp"

### Liên tưởng

Giống như quy trình **ký duyệt giấy tờ** trong công ty:
- Nhân viên (hoặc robot) gửi đơn xin phê duyệt
- Đơn được giao cho người có thẩm quyền
- Người duyệt có thể: Duyệt ✅, Từ chối ❌, hoặc để quá hạn ⏰
- Một số công việc bị **chặn** cho đến khi đơn được duyệt

### Bước 1: Tạo yêu cầu phê duyệt

```
Ai có thể tạo: User HOẶC Agent

Ví dụ: Agent "DeployBot" muốn deploy lên production:
  → POST /api/v1/approvals {
      title: "Deploy PR #42 to production",
      approval_type: "deployment",
      priority: "high",
      assigned_to_user_id: lead_id,     ← Giao cho ai duyệt
      task_ids: [deploy_task_id],       ← Link tới task nào
      link_type: "blocks",             ← Task bị chặn cho đến khi duyệt
      context: {                       ← Thông tin bổ sung
        pr_url: "github.com/...",
        test_results: "all passing"
      }
    }

Backend:
  1. Tạo Approval (status: "pending")
  2. Tạo ApprovalTaskLink → liên kết approval với deploy_task
  3. Activity: "approval.created"
  4. Webhook: "approval.created"
```

### Bước 2: Người duyệt xem và quyết định

```
Board lead vào /approvals
  → Tab "Pending" hiện danh sách đơn chờ duyệt
  → ApprovalCard hiển thị:
    ┌──────────────────────────────────────────┐
    │ 🔴 HIGH  Deploy PR #42 to production     │
    │ Requested by: DeployBot (agent)          │
    │ Linked tasks: 1 task blocked             │
    │ Context: PR #42, tests all passing       │
    │                                          │
    │         [✅ Approve]  [❌ Reject]         │
    └──────────────────────────────────────────┘

Duyệt:
  → POST /api/v1/approvals/{id}/approve { note: "LGTM, ship it" }
  → approval.status = "approved"
  → Linked task được UNBLOCK:
    - task.status "blocked" → "todo" (sẵn sàng thực hiện)
  → Activity: "approval.approved"
  → Agent thấy approval đã duyệt → tiến hành deploy

Từ chối:
  → POST /api/v1/approvals/{id}/reject { note: "Cần thêm test" }
  → approval.status = "rejected"
  → Task vẫn bị block
  → Agent thấy từ chối → xử lý feedback
```

### Bước 3: Tự động hết hạn

```
Nếu approval có expires_at và quá thời gian:
  → Job định kỳ chạy check_expired_approvals()
  → Tìm approval pending + expires_at < now
  → Chuyển status = "expired"
  → Task vẫn bị block (cần tạo approval mới)
```

### Sơ đồ trạng thái Approval

```
              ┌─────────┐
              │ pending  │
              └────┬─────┘
                   │
         ┌────────┼────────┐
         ▼        ▼        ▼
    ┌────────┐ ┌────────┐ ┌─────────┐
    │approved│ │rejected│ │cancelled│
    └────────┘ └────────┘ └─────────┘
                          (requester huỷ)
         │
    quá hạn?
         ▼
    ┌────────┐
    │expired │
    └────────┘

Khi approved + link_type="blocks":
  → Linked tasks tự động unblock
```

---

## Flow 5: Gateway — "Cổng kết nối thế giới bên ngoài"

### Liên tưởng

Gateway giống như **bến xe** nơi các agent (xe buýt) đỗ và được quản lý:
- Gateway là hệ thống bên ngoài chạy các agent
- Mission Control không trực tiếp chạy agent, mà **đăng ký và giám sát** qua gateway
- Một gateway có thể quản lý nhiều agent

### Bước 1: Thêm Gateway

```
Admin vào /gateways → "Add Gateway"
  → Form: tên, endpoint URL, API key, loại (openclaw/custom)

POST /api/v1/gateways {
  name: "Production Gateway",
  endpoint_url: "https://gateway.example.com/api",
  api_key: "gw_sk_...",
  gateway_type: "openclaw"
}

→ Tạo Gateway record (status: "unknown")
```

### Bước 2: Health Check — "Gateway có sống không?"

```
POST /api/v1/gateways/{id}/health-check

Backend:
  1. Gọi GET {endpoint_url}/health (timeout 10s)
  2. Response 200 → status = "connected" ✅
  3. Timeout → status = "disconnected" 🔌
  4. Lỗi khác → status = "error" ❌
  5. Cập nhật last_health_check_at

Frontend hiện badge:
  🟢 Connected (xanh lá)
  🔴 Error (đỏ)
  ⚪ Unknown (xám)
```

### Bước 3: Provision Agent qua Gateway

```
Khi tạo agent với gateway_id:
  → provision_agent() gọi POST {gateway_url}/agents/provision
    Body: { agent_name, agent_type, capabilities, config }
  → Gateway trả về gateway_agent_id (ID trong hệ thống gateway)
  → Lưu vào agent.gateway_agent_id
  → agent.provisioned_at = now

Khi xoá agent:
  → deprovision_agent() gọi DELETE {gateway_url}/agents/{gateway_agent_id}
  → Xoá agent khỏi gateway
```

```
  Mission Control                    Gateway
  ┌──────────────┐                ┌──────────────┐
  │              │  POST /provision│              │
  │  Agent record├───────────────►│ Tạo runtime  │
  │  (DB)        │◄───────────────┤ cho agent    │
  │              │  gateway_agent_id              │
  │              │                │              │
  │              │  GET /health   │              │
  │              ├───────────────►│  {"ok":true} │
  │              │◄───────────────┤              │
  └──────────────┘                └──────────────┘
```

---

## Flow 6: Webhook — "Chuông báo tự động"

### Liên tưởng

Webhook giống như **dịch vụ gửi thông báo tự động**:
- Bạn đăng ký: "Khi có task mới trên board X, gửi tin nhắn vào Slack"
- Hệ thống tự động gửi mỗi khi sự kiện xảy ra
- Nếu gửi thất bại nhiều lần → tự động tắt (tránh spam)

### Bước 1: Cấu hình Webhook

```
Board admin → Board Settings → Webhooks → "Add Webhook"
  → URL: "https://hooks.slack.com/..."
  → Secret: "whsec_abc123" (tuỳ chọn, dùng để xác minh)
  → Events: ☑ task.created  ☑ task.completed  ☑ approval.created
  → Bấm "Save"

POST /api/v1/board-webhooks {
  board_id, url, secret, events: ["task.created", "task.completed", ...]
}
```

### Bước 2: Test Webhook — "Thử chuông kêu không"

```
POST /api/v1/board-webhooks/{id}/test

Backend gửi payload test:
{
  "event": "webhook.test",
  "timestamp": "2024-01-01T00:00:00Z",
  "data": { "message": "This is a test webhook delivery" }
}

→ Trả về: ✅ status_code: 200  hoặc  ❌ error
```

### Bước 3: Webhook tự động kích hoạt

```
Khi user tạo task trên board:
  1. create_task() gọi dispatch_webhook()
  2. Tìm tất cả webhook của board đang active + subscribe "task.created"
  3. Với mỗi webhook khớp:
     → Đưa vào hàng đợi RQ (Redis Queue) — KHÔNG gửi trực tiếp
     → Tránh block API response

RQ Worker xử lý (process riêng):
  1. Lấy webhook config từ DB
  2. Build payload:
     {
       "event": "task.created",
       "timestamp": "...",
       "board_id": "...",
       "data": { ...task_data }
     }
  3. Nếu có secret → tạo HMAC-SHA256 signature
  4. POST tới webhook URL kèm headers:
     - X-Webhook-Event: "task.created"
     - X-Webhook-Signature-256: "sha256=..." (nếu có secret)
     - X-Webhook-Delivery-Id: "<uuid>" (ID giao hàng)
  5. Ghi nhận kết quả

Retry khi thất bại:
  → Lần 1: đợi 10 giây
  → Lần 2: đợi 1 phút
  → Lần 3: đợi 5 phút
  → Sau 10 lần thất bại liên tiếp → TỰ ĐỘNG TẮT webhook
```

```
  Board Event ──► dispatch_webhook() ──► Redis Queue ──► RQ Worker
                                                            │
                                                 ┌──────────┼──────────┐
                                                 ▼          ▼          ▼
                                            Slack Hook  Discord Hook  Custom API

  Retry: 10s → 1min → 5min
  Auto-disable sau 10 failures liên tiếp
```

### HMAC Signing — "Tem chống giả"

```
Tại sao cần secret?
  → Đảm bảo webhook thực sự đến từ Mission Control
  → Tránh kẻ xấu gửi webhook giả

Cách hoạt động:
  1. Body = JSON payload
  2. Signature = HMAC-SHA256(secret, body)
  3. Gửi kèm header: X-Webhook-Signature-256: sha256=<signature>

Bên nhận (Slack/custom):
  1. Nhận body + signature
  2. Tự tính: expected = HMAC-SHA256(shared_secret, body)
  3. So sánh: signature === expected?
  → Khớp = webhook thật ✅
  → Không khớp = webhook giả ❌
```

---

## Flow 7: Onboarding — "Hướng dẫn người mới"

### Liên tưởng

Giống như khi bạn **mở hộp điện thoại mới** — có tờ "Quick Start Guide" hướng dẫn từng bước: bật máy, đăng nhập, cài app đầu tiên...

### Flow chi tiết

```
User vừa tạo board mới → vào board detail
  → Frontend gọi GET /api/v1/board-onboarding/{board_id}/status

Backend tính toán:
  ✅ Bước 1: Tạo board                → Luôn completed (bạn đã ở đây)
  ⬜ Bước 2: Cấu hình columns         → column_config != null && length > 0?
  ⬜ Bước 3: Tạo task đầu tiên        → Board có ít nhất 1 task?
  ⬜ Bước 4: Mời team member           → Board có >= 2 members?
  ⬜ Bước 5: Gán agent                 → Board có agent nào không?
  ⬜ Bước 6: Cấu hình webhook (tuỳ chọn) → Board có webhook?

Trả về:
{
  steps: [...],
  completed_count: 1,
  total_required: 5,
  is_complete: false
}
```

**Giao diện:**
```
┌─────────────────────────────────────────┐
│  🚀 Get Started with "Sprint 42"       │
│                                         │
│  ████░░░░░░░░░░░░░░░░  1/5 completed   │
│                                         │
│  ✅ Create your board                   │
│  ⬜ Configure columns         [→ Set up]│
│  ⬜ Create your first task    [→ Create]│
│  ⬜ Invite a team member      [→ Invite]│
│  ⬜ Assign an agent           [→ Assign]│
│  ⬜ Set up webhooks (optional)[→ Setup] │
│                                         │
│                    [Skip for now]        │
└─────────────────────────────────────────┘
```

**Mỗi bước hoàn thành:**
- Frontend refetch onboarding status
- Backend kiểm tra dữ liệu thực (có task thật không? có member thật không?)
- Progress bar cập nhật

**Dismiss:**
```
User bấm "Skip for now"
  → POST /api/v1/board-onboarding/{board_id}/dismiss
  → board.config.onboarding_dismissed = true
  → Banner biến mất
```

---

## Flow 8: Activity Feed — "Nhật ký mọi chuyện"

### Liên tưởng

Activity Feed giống như **camera giám sát** ghi lại mọi hành động trong toà nhà:
- Ai vào phòng nào, lúc nào
- Ai di chuyển đồ vật (task) từ đâu tới đâu
- Robot nào nhận việc, hoàn thành, hay gặp lỗi
- Ai phê duyệt cái gì

### Cách ghi nhận sự kiện

```
Mọi hành động quan trọng đều gọi record_activity():

async def record_activity(
    session, event_type, actor,
    board_id?, task_id?, agent_id?, approval_id?,
    metadata?, summary?
)

Ví dụ:
  await record_activity(session, "task.status_changed",
    actor=auth,
    task_id=task.id,
    board_id=board.id,
    metadata={"old_status": "todo", "new_status": "in_progress"}
  )

Tự động:
  - category = event_type.split(".")[0]  → "task"
  - actor_type = auth.actor_type         → "user" hoặc "agent"
  - summary = tự generate nếu không truyền
```

### Danh sách event types

```
Board:
  board.created      │ Board được tạo
  board.updated      │ Board settings thay đổi
  board.archived     │ Board bị archive

Task:
  task.created        │ Task mới được tạo
  task.updated        │ Task fields thay đổi
  task.status_changed │ Task chuyển cột (kèm old/new status)
  task.claimed        │ Agent nhận task
  task.completed      │ Task hoàn thành

Agent:
  agent.created       │ Agent được đăng ký
  agent.activated     │ Agent được bật lại
  agent.deactivated   │ Agent bị tắt
  agent.went_offline  │ Agent mất heartbeat

Approval:
  approval.created    │ Yêu cầu phê duyệt mới
  approval.approved   │ Được duyệt
  approval.rejected   │ Bị từ chối

Member:
  member.joined       │ User mới tham gia board
```

### Xem Activity

```
Toàn tổ chức:
  GET /api/v1/activity?limit=50
  → Tất cả events gần nhất

Theo board:
  GET /api/v1/activity?board_id=xxx
  → Chỉ events trên board đó

Theo task:
  GET /api/v1/activity?task_id=xxx
  → Lịch sử của 1 task cụ thể

Theo agent:
  GET /api/v1/activity?agent_id=xxx
  → Mọi hành động của agent

Timeline view:
  GET /api/v1/activity/timeline?days=7
  → Events nhóm theo ngày (7 ngày gần nhất)
```

**Giao diện Timeline:**
```
┌─────────────────────────────────────────────────────┐
│  📋 Activity Feed                [Filter ▾] [7 days]│
│                                                     │
│  ── Today ──────────────────────────────────────── │
│                                                     │
│  🟢 Task "Fix login bug" created by John      2m   │
│     on Board Alpha                                  │
│                                                     │
│  🔄 Agent "CodeBot" claimed "Update API docs" 15m  │
│                                                     │
│  ✅ Task "Deploy v2.1" completed by DeployBot  1h  │
│                                                     │
│  ── Yesterday ─────────────────────────────────── │
│                                                     │
│  ⚠️ Approval "Prod deploy" pending review      2h  │
│                                                     │
│  👤 Sarah joined Board Beta                    3h  │
│                                                     │
│  🤖 Agent "CodeBot" went offline               5h  │
│                                                     │
│                    [Load more...]                    │
└─────────────────────────────────────────────────────┘
```

---

## Tổng kết: Cách các flow liên kết với nhau

```
                         ┌──────────────┐
                         │     Auth     │
                         │  (Cổng vào)  │
                         └──────┬───────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
              ┌──────────┐ ┌────────┐ ┌─────────┐
              │  Boards  │ │ Agents │ │Gateways │
              │(Bàn cờ)  │ │(Robot) │ │ (Bến xe)│
              └────┬─────┘ └───┬────┘ └────┬────┘
                   │           │            │
                   ▼           ▼            │
              ┌──────────┐    claim         │
              │  Tasks   │◄────────provision│
              │(Quân cờ) │                  │
              └────┬─────┘
                   │
         ┌────────┼────────┐
         ▼        ▼        ▼
    ┌─────────┐ ┌──────┐ ┌──────────┐
    │Approvals│ │Webhks│ │ Activity │
    │(Phê duyệt)│(Chuông)│(Nhật ký) │
    └─────────┘ └──────┘ └──────────┘

Mọi hành động → ghi Activity
Mọi sự kiện quan trọng → trigger Webhook
Task bị block → cần Approval
Agent nhận task → cập nhật Board
Gateway quản lý Agent runtime
Onboarding hướng dẫn setup Board ban đầu
```

**Luồng chính một ngày làm việc:**
```
1. User login (Auth) → vào Dashboard
2. Tạo Board + Onboarding hướng dẫn setup
3. Tạo Tasks trên Board
4. Register Agents, assign vào Board
5. Agents claim tasks, làm việc
6. Agent cần approve → tạo Approval → Lead duyệt
7. Task hoàn thành → Webhook gửi Slack
8. Mọi thứ ghi vào Activity Feed
9. Board Snapshot chụp tiến độ cuối ngày
```
