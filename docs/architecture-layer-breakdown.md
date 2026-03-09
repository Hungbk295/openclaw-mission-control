# OpenClaw Mission Control — Architecture Layer Breakdown

## Bài toán gốc và tư duy thiết kế

OpenClaw Mission Control giải quyết một bài toán rõ ràng: khi bạn vận hành nhiều AI agent cùng lúc, trên nhiều board, thuộc nhiều tổ chức khác nhau — bạn cần một nơi duy nhất để nhìn thấy mọi thứ đang diễn ra, kiểm soát ai được làm gì, và can thiệp khi cần.

Từ bài toán đó, kiến trúc hệ thống được xây dựng thành **5 layer** xếp chồng lên nhau, mỗi layer giải quyết đúng một mối quan tâm, và layer trên chỉ giao tiếp với layer ngay dưới nó.

---

## Layer 1: Data & Persistence

**Mối quan tâm:** Dữ liệu được lưu trữ và truy vấn như thế nào?

Nền tảng là **PostgreSQL** (async), truy cập qua **SQLModel** — một thư viện kết hợp Pydantic (validation) và SQLAlchemy (ORM) trong cùng một class. Lý do chọn SQLModel thay vì SQLAlchemy thuần: khi model vừa là ORM entity vừa là Pydantic schema, bạn không phải viết mapper giữa hai thế giới. Một class `Board` vừa map được vào bảng database, vừa serialize thẳng ra JSON response.

**QueryModel pattern:** Mọi model đều kế thừa từ `QueryModel` — một base class gắn sẵn `ManagerDescriptor`. Descriptor này cho phép viết query theo chuỗi giống Django ORM:

```python
board = await Board.objects.by_id(board_id).first(session)
agents = await Agent.objects.filter_by(board_id=board.id).all(session)
```

Tại sao không dùng raw SQLAlchemy query? Vì team muốn tách biệt **cách viết query** khỏi **cách lấy session**. `QuerySet` là immutable — mỗi lần gọi `.filter()` trả về một QuerySet mới, không mutate cái cũ. Điều này an toàn cho async context và dễ compose.

**Multi-tenancy:** `TenantScoped` mixin đánh dấu model nào thuộc phạm vi tổ chức (organization). Đây là convention, không phải magic — nhưng nó buộc mọi developer khi tạo model mới phải tự hỏi: "entity này thuộc về một organization hay là global?"

**Migration:** Alembic quản lý schema migration. Không có gì đặc biệt ở đây — nhưng đáng chú ý là migration được tách riêng khỏi app code (`backend/migrations/`), cho phép rollback database độc lập với rollback code.

---

## Layer 2: Service & Business Logic

**Mối quan tâm:** Logic nghiệp vụ và orchestration nằm ở đâu?

Đây là layer dày nhất và cũng là nơi thiết kế thể hiện rõ quan điểm nhất. Team quyết định **API routes phải mỏng** — route chỉ parse request, gọi service, trả response. Mọi logic phức tạp nằm trong `backend/app/services/`.

Có 3 nhóm service chính:

### a) Domain services (CRUD mở rộng)

`activity_log.py`, `organizations.py`, `tags.py`, `task_dependencies.py` — các service này wrap CRUD operations với business rules. Ví dụ, `task_dependencies.py` không chỉ tạo dependency record, mà còn validate dependency graph để tránh circular reference.

### b) OpenClaw integration services (lõi điều phối)

Đây là phần phức tạp nhất và cũng là lý do project này tồn tại. Nằm trong `backend/app/services/openclaw/`, gồm:

- **`gateway_dispatch.py`** — resolve gateway config cho board và gửi message đến agent session. Đây là low-level: "tôi có một gateway config và một session key, hãy gửi message đi."

- **`gateway_rpc.py`** — client để gọi OpenClaw gateway qua JSON-RPC. Tại sao JSON-RPC thay vì REST? Vì gateway operations không map tự nhiên vào REST verbs — "nudge agent", "read soul file", "ensure session" là commands, không phải resources.

- **`coordination_service.py`** — orchestration layer cao nhất. Nó kết nối các khái niệm: board, agent, gateway, lead/worker hierarchy. Ví dụ, khi gateway-main agent cần hỏi user một câu, flow là: lead agent → `GatewayCoordinationService.ask_user_via_gateway_main()` → compose structured message → dispatch tới gateway-main session → gateway-main chuyển câu hỏi đến user qua Slack/SMS.

- **`provisioning_db.py`** — quản lý lifecycle của agent: tạo, gán board, gán gateway, ensure lead agent tồn tại. Tách ra khỏi coordination vì provisioning là idempotent (gọi nhiều lần cho cùng kết quả), còn coordination thì không.

- **`policies.py`** (`OpenClawAuthorizationPolicy`) — tập trung authorization rules cho gateway operations. Tại sao tách policies ra class riêng thay vì để trong service? Vì authorization logic cần dễ audit — khi security review, bạn chỉ cần đọc một file thay vì grep cả codebase.

**Retry pattern:** Gateway calls dùng `with_coordination_gateway_retry()` — exponential backoff cho transient failures. Điều này phản ánh thực tế vận hành: gateway có thể tạm thời unavailable, và retry đúng cách rẻ hơn nhiều so với fail ngay.

### c) Queue-based async processing

**Redis + RQ** xử lý webhook inbound: khi external system POST webhook, payload được lưu DB ngay lập tức (để không mất data), rồi enqueue vào Redis để worker xử lý async. Pattern này tách biệt **nhận** (phải nhanh, phải reliable) khỏi **xử lý** (có thể chậm, có thể retry).

---

## Layer 3: API & Access Control

**Mối quan tâm:** Ai được gọi endpoint nào, với quyền gì?

### API Design

Backend dùng **FastAPI** với prefix `/api/v1`. Router tổ chức theo domain entity: `boards.py`, `tasks.py`, `agents.py`, `approvals.py`... Mỗi router file tương ứng với một nhóm endpoints liên quan.

**Pagination** dùng limit-offset (max 200 items), trả metadata qua headers (`X-Total-Count`, `X-Limit`, `X-Offset`). Tại sao không cursor-based? Vì dữ liệu ở đây (boards, tasks, agents) thay đổi chậm và users cần jump to page — limit-offset đơn giản hơn và đủ dùng cho scale này.

### Access Control (deps.py — trung tâm policy)

Đây là file quan trọng nhất về mặt security. Team chọn **FastAPI Dependency Injection** làm cơ chế enforce authorization — mọi permission check được compose từ các dependency reusable:

```
require_admin_auth()      → chỉ admin user
require_admin_or_agent()  → admin user HOẶC authenticated agent
require_org_member()      → phải là member của organization
get_board_for_actor_read()  → load board + check read permission
get_board_for_actor_write() → load board + check write permission
get_task_or_404()         → load task + verify nó thuộc board đúng
```

Tại sao thiết kế theo kiểu compose dependencies thay vì decorator hay middleware?

1. **Type-safe:** FastAPI DI tự inject đúng type, IDE hiểu được.
2. **Composable:** `get_task_or_404` depend vào `BOARD_READ_DEP`, tự động chain check board access trước khi check task.
3. **Testable:** Mock một dependency = mock cả chain permission check.
4. **Auditable:** Nhìn vào function signature của endpoint là biết nó yêu cầu quyền gì.

**Dual actor model:** Hệ thống có hai loại caller — human user và AI agent. `ActorContext` abstract hóa sự khác biệt này. Route dùng `require_admin_or_agent()` sẽ chấp nhận cả hai, nhưng apply authorization rules khác nhau (agent bị giới hạn trong board của nó).

### Authentication

Hai mode: **Clerk** (JWT, cho production multi-tenant) và **Local** (bearer token, cho self-hosted). Cùng một codebase, cùng một flow — chỉ khác cách resolve user identity từ token.

### Real-time: Server-Sent Events (SSE)

Thay vì WebSocket, team chọn **SSE** cho streaming updates (agent heartbeats, task changes, activity feed). Tại sao?

- SSE đơn giản hơn WebSocket — chỉ cần HTTP, không cần upgrade protocol.
- Data flow là unidirectional (server → client) — client không cần push data lên qua socket.
- Hoạt động tốt qua load balancer và proxy mà không cần cấu hình đặc biệt.
- Fallback tự nhiên: nếu SSE disconnect, client chỉ cần reconnect với `since` timestamp.

Pattern: backend poll database mỗi 2 giây, dedup bằng seen set, stream JSON events qua `EventSourceResponse`.

---

## Layer 4: Frontend Application

**Mối quan tâm:** User nhìn thấy gì và tương tác như thế nào?

### Framework & Routing

**Next.js** (App Router) với file-based routing. Cấu trúc route phản ánh domain model:

```
/boards/[boardId]        → board detail + tasks
/agents/[agentId]        → agent detail + health
/approvals               → pending approval queue
/gateways/[gatewayId]    → gateway config + connected boards
/activity                → audit timeline
/settings                → org settings
/skills                  → skill marketplace
```

### API Client Generation (Orval)

Đây là design decision quan trọng: **không viết API client bằng tay**. Backend expose OpenAPI schema, **Orval** đọc schema đó và generate:

- TypeScript types cho mọi request/response
- React Query hooks cho mọi endpoint (useQuery, useMutation)
- Query key factories cho cache invalidation

Tại sao? Vì khi backend thay đổi schema, frontend chỉ cần chạy `make api-gen` — type errors sẽ xuất hiện ngay ở compile time thay vì runtime. Zero hand-written fetch code = zero chance viết sai URL hay miss field.

### State Management

**Không dùng Redux hay Zustand.** Server state được quản lý hoàn toàn bằng **React Query** (TanStack Query v5). Lý do:

- 95% state trong app này là server state (boards, tasks, agents) — không phải client-only state.
- React Query đã handle caching, refetching, optimistic updates, loading/error states.
- Thêm Redux chỉ tạo ra duplication: store phải sync với server, query hook cũng sync với server → hai nguồn truth.

### HTTP Client (`customFetch`)

Một wrapper duy nhất cho mọi API call. Nó:
1. Resolve base URL từ env
2. Auto-attach auth token (Clerk JWT hoặc local bearer)
3. Normalize response thành `{ data, status, headers }`
4. Parse error detail từ FastAPI validation response format
5. Handle SSE stream response riêng

Mọi generated hook đều đi qua `customFetch` — developer không bao giờ gọi `fetch()` trực tiếp.

### UI Layer

**Radix UI** (unstyled primitives) + **Tailwind CSS**. Tại sao không dùng component library có sẵn style (MUI, Ant Design)?

- Radix cho accessibility out-of-the-box (keyboard nav, ARIA, focus management) mà không ép style.
- Tailwind cho phép custom design system mà không phải override framework CSS.
- Kết hợp hai thứ này = accessible components + custom look, không bị khóa vào aesthetic của framework.

Các thành phần bổ sung: **Recharts** cho charts/metrics, **react-markdown** với GFM cho rich text rendering (agent messages, task descriptions), **TanStack Table** cho data tables có sort/filter/pagination.

---

## Layer 5: Infrastructure & Deployment

**Mối quan tâm:** Hệ thống chạy ở đâu và kết nối với nhau như thế nào?

### Docker Compose

Full stack chạy trong Docker Compose: PostgreSQL, Redis, backend (FastAPI + Uvicorn), frontend (Next.js), worker (RQ). `compose.yml` + `.env` là đủ để boot toàn bộ.

Tại sao không Kubernetes? Vì target audience là "platform teams running OpenClaw in self-hosted environments" — Docker Compose đủ cho self-hosted deployment, giảm ops overhead so với K8s.

### External Dependencies

```
PostgreSQL ──── persistent storage (boards, tasks, agents, orgs)
Redis ───────── job queue (webhook processing via RQ)
Clerk ───────── auth provider (optional, thay bằng local token)
OpenClaw Gateway ── agent runtime (JSON-RPC communication)
```

### Gateway Communication Model

Mission Control không chạy agent trực tiếp. Nó giao tiếp với **OpenClaw Gateway** — một runtime riêng quản lý agent sessions. Communication qua JSON-RPC, với patterns:

- `ensure_session` → đảm bảo agent có session trên gateway
- `send_message` → gửi message vào session (có thể deliver ngay hoặc queue)
- `agents.files.get/set` → đọc/ghi file trên agent (ví dụ SOUL.md)

Tại sao tách gateway ra khỏi Mission Control? Vì agent runtime có lifecycle và scaling requirements khác với control plane. Gateway có thể chạy ở edge, Mission Control chạy centralized — separation of concerns ở infrastructure level.

---

## Luồng dữ liệu end-to-end (ví dụ minh họa)

**Scenario: User tạo task trên board, agent nhận và xử lý**

```
[Browser]
  → React component gọi useCreateTask() (generated hook)
    → customFetch() attach JWT, POST /api/v1/boards/{id}/tasks
      → FastAPI route: deps resolve auth + board access
        → Service layer: create task, record activity
          → DB: INSERT task, INSERT activity_event
        → Response: TaskRead schema
      → React Query: cache update, UI re-render

[SSE Stream]
  → Agent đang listen GET /boards/{id}/tasks/stream
    → Backend poll DB mỗi 2s, phát hiện task mới
      → Stream JSON event đến agent
        → Agent xử lý task, update status
          → PATCH /api/v1/boards/{id}/tasks/{id}
            → DB: UPDATE task status
              → SSE lại notify browser
                → UI cập nhật real-time
```

---

## Tổng kết: Tại sao kiến trúc này hoạt động

Mỗi quyết định thiết kế đều xuất phát từ cùng một nguyên tắc: **giữ mỗi layer đơn giản bằng cách đẩy complexity xuống đúng chỗ**.

- Database layer đơn giản vì QuerySet pattern abstract query building.
- Service layer chứa complexity vì đó là nơi business logic thuộc về.
- API layer đơn giản vì deps.py tập trung authorization, routes chỉ wire things together.
- Frontend đơn giản vì Orval generate toàn bộ API client, React Query quản lý state.
- Infrastructure đơn giản vì Docker Compose, gateway tách biệt.

Không layer nào cố làm quá nhiều việc. Không layer nào bị leak concern từ layer khác. Đó là lý do hệ thống vận hành được ở quy mô multi-org, multi-agent mà codebase vẫn navigable.
