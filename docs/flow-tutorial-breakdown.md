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
- Đơn được giao cho người có thẩm quyền (board lead)
- Người duyệt có thể: Duyệt ✅ hoặc Từ chối ❌
- Mỗi task chỉ có **tối đa 1 approval pending** tại một thời điểm
- Khi quyết định xong → board lead agent được **tự động thông báo** qua gateway

### Bước 1: Tạo yêu cầu phê duyệt

```
Ai có thể tạo: User HOẶC Agent

Ví dụ: Agent "CodeBot" muốn deploy lên production:
  → POST /api/v1/boards/{board_id}/approvals {
      action_type: "deploy_to_production",
      task_ids: [deploy_task_id],         ← Link tới task nào (1 hoặc nhiều)
      agent_id: agent_id,                 ← Agent nào yêu cầu
      confidence: 0.95,                   ← Độ tự tin của agent
      rubric_scores: {...},               ← Điểm đánh giá (tuỳ chọn)
      payload: {                          ← Thông tin bổ sung
        pr_url: "github.com/...",
        test_results: "all passing"
      }
    }

Backend (create approval):
  1. Normalize task IDs (dedupes, merges task_id + task_ids + payload.taskId)
  2. Lock tasks theo thứ tự deterministic (tránh deadlock)
  3. Kiểm tra: task này đã có pending approval chưa?
     → Nếu có → 409 Conflict
  4. Tạo Approval record (status: "pending")
  5. Tạo ApprovalTaskLink rows (many-to-many)
  6. Ghi activity event
```

### Bước 2: Người duyệt xem và quyết định

```
User vào /approvals (cross-board view, polls mỗi 15 giây)
  → Hiện danh sách đơn chờ duyệt với task titles

  Hoặc dùng SSE real-time:
  → GET /api/v1/boards/{board_id}/approvals/stream
  → Server push mỗi 2 giây nếu có thay đổi
  → Gửi: approval data + pending_approvals_count + task-level counts

Duyệt:
  → PATCH /api/v1/boards/{board_id}/approvals/{id} { status: "approved" }
  → approval.status = "approved", resolved_at = now
  → Board lead agent được THÔNG BÁO tự động:
    → GatewayDispatchService gửi message tới lead's openclaw_session_id
    → Nội dung: board name, approval ID, action type, decision, task IDs
    → Activity: "approval.lead_notified"
  → Agent thấy approval đã duyệt → tiến hành công việc

Từ chối:
  → PATCH /api/v1/boards/{board_id}/approvals/{id} { status: "rejected" }
  → Board lead agent cũng được thông báo rejection
  → Agent xem feedback, điều chỉnh
```

### Sơ đồ trạng thái Approval

```
              ┌─────────┐
              │ pending  │
              └────┬─────┘
                   │
         ┌────────┴────────┐
         ▼                 ▼
    ┌────────┐        ┌────────┐
    │approved│        │rejected│
    └────┬───┘        └────────┘
         │
         ▼
  Board lead agent
  tự động nhận thông báo
  qua GatewayDispatchService

Ràng buộc:
  - Mỗi task chỉ có 1 pending approval tại 1 thời điểm
  - Task IDs locked theo thứ tự deterministic (chống deadlock)
  - Optimistic UI updates trên frontend
```

### Approval & Board require_approval_for_done

```
Board có setting: require_approval_for_done (mặc định = true)
  → Khi agent muốn đánh dấu task "done"
  → Phải tạo approval trước
  → User duyệt → task mới thực sự done

Ngoại lệ: Nếu board onboarding detect autonomy_level = "fully autonomous"
  → require_approval_for_done = false
  → Agent tự do đánh dấu done không cần duyệt
```

---

## Flow 5: Gateway — "Cổng kết nối thế giới bên ngoài"

### Liên tưởng

Gateway giống như **nhà máy** nơi các agent (robot) thực sự chạy:
- Mission Control là **trung tâm điều khiển** (ra lệnh, giám sát)
- Gateway là **sàn sản xuất** (nơi robot hoạt động thực tế)
- Gateway kết nối qua **WebSocket** (ws:// hoặc wss://)
- Mỗi gateway tự động có 1 **Main Agent** (giám đốc nhà máy)

### Bước 1: Thêm Gateway

```
Org admin vào /gateways/new
  → Form: tên, WebSocket URL, token, workspace_root

POST /api/v1/gateways {
  name: "Production Gateway",
  url: "wss://gateway.example.com:8080",    ← WebSocket URL (phải có port)
  token: "gw_token_...",
  workspace_root: "/opt/agents"             ← Thư mục gốc trên gateway
}

Backend (GatewayAdminLifecycleService):
  1. Validate gateway runtime compatible (kết nối thử)
  2. Tạo Gateway record
  3. TỰ ĐỘNG tạo Main Agent:
     → ensure_main_agent(action="provision")
     → Main Agent = agent không thuộc board nào
     → Quản lý gateway-level operations
```

**Validation WebSocket URL:**
```
Frontend (validateGatewayUrl):
  ✅ wss://gateway.example.com:8080  (có port, HTTPS)
  ✅ ws://localhost:9090              (có port, dev mode)
  ❌ https://gateway.example.com      (không phải ws/wss)
  ❌ wss://gateway.example.com        (thiếu port)
```

### Bước 2: Kiểm tra kết nối

```
Frontend gọi checkGatewayConnection():
  → POST /api/v1/gateways/status
  → Backend thử kết nối WebSocket tới gateway
  → Trả về trạng thái kết nối
```

### Bước 3: Template Sync — "Cập nhật tài liệu cho tất cả robot"

```
POST /api/v1/gateways/{gateway_id}/templates/sync
  Query params:
    include_main: true/false       ← Sync cả main agent?
    reset_sessions: true/false     ← Reset agent sessions?
    rotate_tokens: true/false      ← Tạo token mới cho tất cả agent?
    force_bootstrap: true/false    ← Buộc chạy lại bootstrap?
    overwrite: true/false          ← Ghi đè file editable?
    lead_only: true/false          ← Chỉ sync board leads?
    board_id: uuid                 ← Chỉ sync 1 board?

Trả về: { agents_updated, agents_skipped, main_updated, errors[] }
```

Đây là tính năng mạnh: cập nhật hàng loạt template files cho tất cả agent trên gateway.

### Bước 4: Xoá Gateway

```
DELETE /api/v1/gateways/{gateway_id}

Backend:
  1. Xoá Main Agent + duplicate agents
  2. Cascade xoá GatewayInstalledSkill rows
  3. Clear agent foreign keys trước khi xoá
  4. Xoá Gateway record
```

```
  Mission Control                         Gateway (WebSocket)
  ┌──────────────┐                     ┌──────────────────┐
  │              │  ws://connect        │                  │
  │  Gateway     ├────────────────────►│  Agent Runtime   │
  │  record (DB) │                      │                  │
  │              │  RPC: agents.create  │  ┌─ Main Agent  │
  │  Main Agent  ├────────────────────►│  │  (gateway-wide)│
  │              │  RPC: agents.update  │  │               │
  │              ├────────────────────►│  ├─ Board Lead   │
  │  Board Agents│                      │  │  (per board)  │
  │              │  Templates sync      │  │               │
  │              ├────────────────────►│  └─ Workers      │
  │              │                      │     (per board)  │
  └──────────────┘                     └──────────────────┘
```

---

## Flow 6: Webhook — "Hộp thư đến của Board"

### Liên tưởng

Webhook trong Mission Control hoạt động ngược lại so với webhook thông thường:
- Đây là **INBOUND webhook** — hệ thống bên ngoài gửi dữ liệu VÀO Mission Control
- Giống như **hộp thư** của board: bên ngoài gửi thư vào, board lead agent đọc và xử lý
- Mỗi webhook tạo ra một **endpoint URL** riêng biệt để bên ngoài POST dữ liệu vào
- Payload được lưu vào **Board Memory** để agent có thể truy cập

### Bước 1: Tạo Webhook Endpoint

```
Board admin → Webhooks → "Add Webhook"
  → Form: description, enabled, agent_id (tuỳ chọn — ai sẽ nhận thông báo)

POST /api/v1/boards/{board_id}/webhooks {
  description: "GitHub PR notifications",
  enabled: true,
  agent_id: null                ← null = giao cho board lead mặc định
}

Backend trả về:
{
  id: "abc-123",
  endpoint_path: "/api/v1/boards/{board_id}/webhooks/abc-123",
  endpoint_url: "https://your-server.com/api/v1/boards/{board_id}/webhooks/abc-123",
  ...
}
  ⬆ Đây là URL bạn cấu hình ở GitHub/Slack/etc.
```

### Bước 2: Bên ngoài gửi dữ liệu vào

```
GitHub (hoặc bất kỳ service nào) POST tới endpoint URL:
  POST /api/v1/boards/{board_id}/webhooks/{webhook_id}
  Body: { "action": "opened", "pull_request": {...} }

Backend xử lý (trả về 202 ACCEPTED ngay lập tức):
  1. Validate: webhook tồn tại VÀ enabled
  2. Decode body (JSON auto-detect, fallback raw text)
  3. Capture headers (content-type, user-agent, x-* headers)
  4. Lưu BoardWebhookPayload:
     → payload, headers, source_ip, content_type, received_at
  5. Tạo BoardMemory entry (để agent đọc được):
     → Tags: "webhook", "webhook:{webhook_id}", "payload:{payload_id}"
  6. Đưa vào Redis queue (enqueue_webhook_delivery)
     → Nếu queue fail → fallback gửi thông báo đồng bộ
```

### Bước 3: Agent nhận thông báo

```
RQ Worker (flush_webhook_delivery_queue) xử lý:
  1. Dequeue từ Redis
  2. Load webhook/payload/board context từ DB
  3. Xác định agent mục tiêu:
     → webhook.agent_id nếu có
     → HOẶC board lead (is_board_lead=True)
  4. Gửi message qua GatewayDispatchService:
     → try_send_agent_message() tới agent's openclaw_session_id
     → Nội dung: webhook instruction + payload preview + API ref

Agent nhận message:
  → Đọc payload preview
  → Nếu cần chi tiết: GET /api/v1/boards/{board_id}/webhooks/{id}/payloads/{payload_id}
  → Xử lý theo logic của agent (tạo task, update task, etc.)
```

### Retry khi thất bại

```
Retry logic (exponential backoff + jitter):
  Base delay: rq_dispatch_retry_base_seconds * (2 ^ attempts)
  Cap: rq_dispatch_retry_max_seconds
  Jitter: ±10% của base delay
  Max retries: rq_dispatch_max_retries (configurable)

Throttle: rq_dispatch_throttle_seconds giữa mỗi item
```

```
  GitHub/Slack/etc.
       │
       │ POST payload
       ▼
  ┌────────────────────┐
  │ Webhook Endpoint   │  ← 202 ACCEPTED (instant)
  └────────┬───────────┘
           │
     ┌─────┼─────┐
     ▼           ▼
  ┌──────┐  ┌──────────┐
  │ DB   │  │  Redis    │
  │Payload│  │  Queue    │
  │+Memory│  └────┬─────┘
  └──────┘       │
                 ▼
           ┌──────────┐
           │RQ Worker │
           └────┬─────┘
                │
                ▼
  GatewayDispatchService
  → Board lead / target agent
  → "Bạn có thư mới từ GitHub!"
```

### Xem lại Payloads

```
Admin hoặc agent có thể xem lại tất cả payload đã nhận:
  GET /api/v1/boards/{board_id}/webhooks/{id}/payloads
  → Paginated, ordered by received_at DESC
  → Mỗi payload: body, headers, source_ip, received_at

Agent truy cập qua Board Memory:
  GET /api/v1/agent/boards/{board_id}/memory?is_chat=false
  → Filter by tags: "webhook", "webhook:{id}", "payload:{id}"
```

---

## Flow 7: Onboarding — "Phỏng vấn để hiểu bạn"

### Liên tưởng

Onboarding KHÔNG phải checklist tĩnh. Đây là một **cuộc trò chuyện** giữa user và
gateway agent — giống như buổi **phỏng vấn onboarding** khi bạn vào công ty mới:
- Agent hỏi bạn 6-10 câu hỏi về mục tiêu, phong cách làm việc
- Bạn trả lời, agent ghi nhận
- Cuối cùng agent đề xuất cấu hình board + tạo board lead agent

### Bước 1: Bắt đầu Onboarding

```
User tạo board mới → vào /onboarding
  → POST /api/v1/boards/{board_id}/onboarding/start

Backend:
  1. Tạo BoardOnboardingSession (status: "active", messages: [])
  2. Gửi comprehensive prompt tới gateway agent qua GatewayDispatchService
  3. Prompt hướng dẫn agent:
     → Hỏi 6-10 câu hỏi tập trung:
       - 3-6 câu về MỤC TIÊU (board dùng để làm gì?)
       - 1 câu về TÊN AGENT (bạn muốn gọi agent lead là gì?)
       - 2-4 câu về SỞ THÍCH (autonomy level, verbosity, update cadence)
     → Thu thập user profile:
       - preferred_name, pronouns, timezone, notes, context
     → Thu thập lead agent config:
       - name, identity_profile, autonomy_level, verbosity
       - output_format, update_cadence, custom_instructions
```

### Bước 2: Cuộc trò chuyện qua lại

```
Gateway agent gửi câu hỏi:
  → POST /api/v1/boards/{board_id}/onboarding/agent
    Body: { type: "question", content: "Board này dùng để làm gì?" }
  → Append assistant message vào messages[]
  → Frontend hiện câu hỏi cho user

User trả lời:
  → POST /api/v1/boards/{board_id}/onboarding/answer
    Body: { answer: "Quản lý sprint cho team backend" }
  → Append user message vào messages[]
  → Forward answer tới agent qua BoardOnboardingMessagingService
  → Agent hỏi câu tiếp theo...

Lặp lại 6-10 lần cho đến khi agent đủ thông tin.
```

### Bước 3: Agent hoàn thành phỏng vấn

```
Agent gửi kết quả:
  → POST /api/v1/boards/{board_id}/onboarding/agent
    Body: {
      type: "complete",
      draft_goal: {
        board_type: "kanban",
        objective: "Quản lý sprint team backend",
        success_metrics: [...],
        target_date: "2024-06-01",
        user_profile: { preferred_name: "Hùng", timezone: "Asia/Ho_Chi_Minh" },
        lead_agent: {
          name: "BackendBot",
          identity_profile: { role: "Sprint manager", emoji: ":robot:" },
          autonomy_level: "semi-autonomous",
          verbosity: "concise",
          update_cadence: "daily"
        }
      }
    }
  → onboarding.status = "completed"
  → draft_goal lưu vào DB
  → Frontend hiện preview cho user xác nhận
```

### Bước 4: User xác nhận và Provision

```
User review draft → bấm "Confirm"
  → POST /api/v1/boards/{board_id}/onboarding/confirm

Backend:
  1. Extract confirmed values từ draft_goal:
     → board.board_type, objective, success_metrics, target_date
     → board.goal_confirmed = true
     → board.goal_source = "lead_agent_onboarding"

  2. Detect autonomy level:
     → Kiểm tra keywords: "autonomous", "fully-autonomous", "full-autonomy"
     → Nếu fully autonomous: board.require_approval_for_done = false
     → Mặc định: board.require_approval_for_done = true

  3. Apply user_profile:
     → Update user: preferred_name, pronouns, timezone, notes, context

  4. PROVISION BOARD LEAD AGENT:
     → OpenClawProvisioningService tạo lead agent theo draft_goal config
     → Agent được provision trên gateway (3-stage process như Flow 3)

  5. onboarding.status = "confirmed"
```

```
  ┌─ Onboarding Flow ───────────────────────────────────────────┐
  │                                                             │
  │  User ◄──────────── Câu hỏi ──────────── Gateway Agent    │
  │    │                                          ▲             │
  │    └── Trả lời ──────────────────────────────┘             │
  │         (6-10 rounds)                                       │
  │                                                             │
  │  Agent gửi draft_goal ──► User xác nhận ──► Provision Lead │
  │                                                             │
  │  Kết quả:                                                   │
  │    ✅ Board configured (type, objective, metrics)           │
  │    ✅ User profile updated (name, timezone, pronouns)       │
  │    ✅ Board Lead Agent provisioned & ready to work          │
  │    ✅ Autonomy level → require_approval_for_done setting    │
  └─────────────────────────────────────────────────────────────┘
```

---

## Flow 8: Activity Feed — "Nhật ký mọi chuyện"

### Liên tưởng

Activity Feed giống như **camera giám sát** ghi lại mọi hành động trong toà nhà.
Đặc biệt, hệ thống có **Task Comments Stream** — feed real-time của tất cả
comments agent đang viết trên tasks, giống như xem **live chat** của cả đội robot.

### Cách ghi nhận sự kiện

```
Mọi hành động quan trọng đều gọi record_activity():

record_activity(
    session,
    event_type="task.comment",
    message="Đang xử lý bước 3/5...",
    agent_id=agent.id,
    task_id=task.id,
)

Model ActivityEvent:
  - id, event_type, message
  - agent_id, task_id          ← reference IDs
  - created_at
  - Indexed on: event_type, agent_id, task_id (fast queries)
```

### Danh sách event types

```
Task:
  task.comment              │ Agent/user comment trên task
  task.status_changed       │ Task chuyển cột

Agent:
  agent.create.*            │ Agent được tạo (direct, by lead, etc.)
  agent.provision.direct    │ Agent được provision
  agent.update.direct       │ Agent updated & re-provisioned
  agent.heartbeat           │ Heartbeat nhận được
  agent.wakeup.sent         │ Wake message gửi đi
  agent.*.failed            │ Errors logged
  agent.delete.direct       │ Agent bị xoá

Approval:
  approval.lead_notified    │ Lead agent nhận thông báo duyệt
  approval.lead_notify_failed│ Thông báo lead thất bại
```

### Xem Activity — 3 cách

**Cách 1: REST API (polling)**
```
GET /api/v1/activity
  → Tất cả events gần nhất
  → Agent chỉ thấy events của mình
  → User thấy events trên boards mình có access
  → Paginated, ordered by created_at DESC
```

**Cách 2: Task Comments Feed (enriched)**
```
GET /api/v1/activity/task-comments
  → Chỉ event_type == "task.comment" có message
  → Join với: Task (title) + Board (name) + Agent (name, role)
  → Agent role lấy từ identity_profile.role
  → Optional filter: board_id
  → Response: ActivityTaskCommentFeedItemRead {
      event_id, message, created_at,
      board: { id, name },
      task: { id, title },
      agent: { id, name, role }
    }
```

**Cách 3: SSE Real-time Stream**
```
GET /api/v1/activity/task-comments/stream
  → Server-Sent Events (SSE) — browser mở connection dài
  → Server poll DB mỗi 2 giây cho events mới
  → Dedup: nhớ 2000 event IDs đã gửi (SSE_SEEN_MAX)
  → Filter theo board access của caller
  → Emit: { event: "comment", data: JSON.stringify({comment: {...}}) }
  → Optional: board_id filter

  Giống như:
    EventSource("/api/v1/activity/task-comments/stream")
    → onmessage: { comment: { agent: "CodeBot", message: "Đang fix bug...", task: "Login fix" } }
```

**Giao diện Activity:**
```
┌─────────────────────────────────────────────────────┐
│  Activity Feed                     [Filter ▾]       │
│                                                     │
│  ── Today ──────────────────────────────────────── │
│                                                     │
│  :gear: CodeBot on "Fix login bug"             2m  │
│     "Đang xử lý bước 3/5..."                       │
│                                                     │
│  :robot: DeployBot on "Deploy v2.1"           15m  │
│     "Build passed, deploying to staging..."         │
│                                                     │
│  :white_check_mark: DeployBot completed "Deploy v2.1"          1h  │
│                                                     │
│  ── Yesterday ─────────────────────────────────── │
│                                                     │
│  :bell: Lead notified: approval "Prod deploy"  2h  │
│                                                     │
│  :heartbeat: CodeBot heartbeat received        5h  │
│                                                     │
│                    [Live updates via SSE]            │
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
