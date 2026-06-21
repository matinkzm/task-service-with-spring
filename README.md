# task-service-with-spring

# Task Document: Task Management Platform

**Technology Stack: NestJS + MongoDB + Redis + WebSockets — MVP**

---

## 1) Purpose

Implement a modular backend platform that delivers core functionality for collaborative task management with real-time capabilities.

This service must support:

1. Secure user authentication and account management
2. Project organization
3. Task lifecycle tracking
4. Task assignment
5. Comments and collaboration
6. Real-time updates through WebSockets
7. Efficient data loading via Redis caching

The MVP must be written with clean REST APIs and designed for extensibility toward:

- WebSocket clients
- Attachments
- Search and filtering
- Event streams
- Multi-service integration

---

## 2) Roles and Access Model

### 2.1 Roles (MVP Scope)

Global platform roles:

- `member`: standard authenticated user
- `admin`: global administrator

Project-level roles:

- `member`: participant in project
- `manager`: participant allowed to manage tasks and assignments

### 2.2 Authentication (Authn)

- Use **JWT access tokens**
- All protected APIs require:
  `Authorization: Bearer <token>`
- Tokens are issued after login
- Password-based login only in MVP (OAuth stretch goal later)

### 2.3 Authorization (Authz)

All permissions must be enforced consistently using an `AccessControlService`.

**User management rules**

- `member` can only read/update their own account
- `admin` can list or modify any user
- Disabled users cannot authenticate

**Project rules**

- Only project participants may access a project
- Only `manager` or `admin` can create, assign, or remove tasks
- Only `manager` or `admin` can modify project participants
- A `member` may update only tasks assigned to them

**Message/comment rules**

- Any project participant may comment
- Only comment author or admin may delete comments

---

## 3) Architecture Overview

### 3.1 Recommended Folder Structure

```
src/
  auth/
    auth.module.ts
    controllers/auth.controller.ts
    services/auth.service.ts
    strategies/jwt.strategy.ts
    guards/jwt-auth.guard.ts
    dto/

  users/
    users.module.ts
    controllers/users.controller.ts
    services/users.service.ts
    repositories/users.repo.ts
    schemas/user.schema.ts
    dto/

  projects/
    projects.module.ts
    controllers/projects.controller.ts
    services/projects.service.ts
    repositories/projects.repo.ts
    schemas/project.schema.ts
    dto/

  tasks/
    tasks.module.ts
    controllers/tasks.controller.ts
    services/tasks.service.ts
    repositories/tasks.repo.ts
    schemas/task.schema.ts
    dto/

  comments/
    comments.module.ts
    controllers/comments.controller.ts
    services/comments.service.ts
    repositories/comments.repo.ts
    schemas/comment.schema.ts
    dto/

  realtime/
    realtime.module.ts
    gateways/tasks.gateway.ts
    services/realtime-cache.service.ts
    services/realtime-events.service.ts
    dto/

  shared/
    access-control/
    pagination/
    redis/
    events/
```

Modules must communicate only via dependency injection and shared interfaces.

---

## 4) MongoDB Data Model

All collections require proper timestamps.

### 4.1 User Collection (`users`)

Fields:

- `_id` – ObjectId
- `email` – string (unique, indexed)
- `passwordHash` – string
- `name` – string
- `role` – enum: `member | admin`
- `status` – enum: `active | disabled`
- `createdAt` – Date
- `updatedAt` – Date
- `lastLoginAt` – Date, nullable

Indexes:

```
db.users.createIndex({ email: 1 }, { unique: true })
```

---

### 4.2 Project Collection (`projects`)

Fields:

- `_id` – ObjectId

- `title` – string

- `description` – string, optional

- `participants[]` – array of objects:

  - `userId` – string
  - `role` – enum: `member | manager`
  - `joinedAt` – Date

- `createdByUserId` – string

- `createdAt` – Date

- `updatedAt` – Date

- `lastTaskAt` – Date, nullable

Indexes:

```
db.projects.createIndex({ title: 1 })
db.projects.createIndex({ "participants.userId": 1 })
```

---

### 4.3 Task Collection (`tasks`)

Fields:

- `_id` – ObjectId
- `projectId` – ObjectId, indexed
- `title` – string
- `description` – string
- `status` – enum: `todo | in_progress | done | blocked`
- `priority` – enum: `low | normal | high`
- `assignedToUserId` – string, nullable
- `createdByUserId` – string
- `createdAt` – Date
- `updatedAt` – Date
- `dueDate` – Date, nullable

Indexes:

```
db.tasks.createIndex({ projectId: 1, createdAt: -1 })
db.tasks.createIndex({ assignedToUserId: 1 })
```

---

### 4.4 Comment Collection (`task_comments`)

Fields:

- `_id` – ObjectId
- `taskId` – ObjectId, indexed
- `userId` – string
- `text` – string
- `createdAt` – Date

Indexes:

```
db.task_comments.createIndex({ taskId: 1, createdAt: -1 })
```

---

## 5) Redis Cache Layer (Mandatory in MVP)

### 5.1 Goals

Redis must be used to:

- Cache frequently loaded lists
- Reduce MongoDB read pressure
- Support real-time unread counts
- Demonstrate caching concepts to junior developer

### 5.2 Cache Keys

Recommended namespaces:

```
tasks:list:{projectId}:{cursor?}
projects:list:{userId}
tasks:item:{taskId}
users:item:{userId}
```

### 5.3 Cache TTL

- Lists: 60 seconds
- Single task/project: 120 seconds
- Unread counts: 30 seconds

### 5.4 Cache Responsibilities

Create a `RedisCacheService` wrapper with:

- get()
- set()
- del()
- keys()

Example minimal service interface:

```typescript
@Injectable()
export class RedisCacheService {
  async getJson<T>(key: string): Promise<T | null> {}
  async setJson(key: string, value: any, ttl: number): Promise<void> {}
  async delete(key: string): Promise<void> {}
}
```

### 5.5 Real-Time Invalidation

Whenever a task is created or updated:

- Delete related list cache keys
- Update single item cache
- Notify WebSocket service

This teaches proper cache invalidation discipline.

---

# 6) WebSocket and Real-Time Layer

---

## 6.1 Purpose

The junior developer must implement:

- A NestJS WebSocket Gateway
- Redis-backed event broadcasting
- Cache invalidation triggered by events

---

## 6.2 Gateway Responsibilities

Create a `TasksGateway` using `@nestjs/websockets`.

Events that must be broadcast:

- `task.created`
- `task.updated`
- `task.assigned`
- `task.status_changed`
- `comment.created`

---

## 6.3 WebSocket Authorization

- Require JWT authentication for socket connections
- Validate token on `handleConnection`
- Reject unauthenticated connections

Example concept:

```typescript
@WebSocketGateway({ namespace: "/tasks" })
export class TasksGateway {
  async handleConnection(client: Socket) {
    // validate JWT token here
  }
}
```

---

## 6.4 Event Flow Design

### Recommended Idea: Event + Cache Pattern

Instead of caching lists blindly, use this pattern:

1. All list endpoints try Redis first
2. On MongoDB change → emit internal event
3. Gateway receives event
4. Gateway invalidates Redis keys
5. Gateway pushes update to clients

This ensures strong consistency.

---

## 6.5 Redis Pub/Sub for WebSockets

Stretch concept inside MVP:

- Use Redis pub/sub channels for cross-instance broadcasting

Example learning concept:

```
redis.publish("tasks.events", payload)
```

The gateway subscribes and forwards events.

This prepares the developer for horizontal scaling.

---

# 7) REST API List (Expanded with Cache + WebSocket)

---

## 7.1 Auth APIs

### A) Register

**POST** `/api/auth/register`

### B) Login

**POST** `/api/auth/login`

### C) Me

**GET** `/api/auth/me`

### D) Logout

**POST** `/api/auth/logout` (optional)

---

## 7.2 Users APIs

### E) Admin: List Users

**GET** `/api/users?limit=25&cursor=...`

### F) Admin: Get User

**GET** `/api/users/:userId`

### G) Update My Profile

**PATCH** `/api/users/me`

### H) Admin: Update User

**PATCH** `/api/users/:userId`

---

## 7.3 Projects APIs

### I) Create Project

**POST** `/api/projects`

### J) List My Projects

**GET** `/api/projects`

### K) Get Project

**GET** `/api/projects/:projectId`

### L) Add Participant

**POST** `/api/projects/:projectId/participants`

### M) Remove Participant

**DELETE** `/api/projects/:projectId/participants/:userId`

---

## 7.4 Task APIs

### N) Create Task

**POST** `/api/projects/:projectId/tasks`

- Invalidate Redis list cache
- Emit `task.created` to WebSocket

---

### O) Update Task

**PATCH** `/api/tasks/:taskId`

Body:

```
{ title?, description?, priority?, dueDate? }
```

Behavior:

- Update `updatedAt`
- Redis:

  - delete cache `tasks:item:{taskId}`
  - invalidate related lists

- Emit `task.updated`

---

### P) Assign Task

**PATCH** `/api/tasks/:taskId/assign`

- Redis invalidate
- WebSocket emit

---

### Q) Update Task Status

**PATCH** `/api/tasks/:taskId/status`

- Same invalidation rules
- Emit `task.status_changed`

---

### R) List Tasks (Cached)

**GET** `/api/projects/:projectId/tasks?limit=50&cursor=...`

Behavior:

1. Attempt Redis key first
2. If cache miss → load from MongoDB
3. Store list in Redis
4. Return paginated list

---

### S) Get Task

**GET** `/api/tasks/:taskId`

- Cached in Redis

---

## 7.5 Comments APIs

### T) Add Comment

**POST** `/api/tasks/:taskId/comments`

- Emit `comment.created`
- Invalidate preview caches

### U) List Comments

**GET** `/api/tasks/:taskId/comments`

### V) Delete Comment (Optional)

**DELETE** `/api/comments/:commentId`

---

## 7.6 Not Implemented in MVP

- Message deletion
- Attachments
- Advanced filtering
- Audit logs

---

# 8) Required Services (Detailed)

---

## 8.1 `AuthService`

Must implement:

- register()
- login()
- validateUser()
- generateJWT()
- update lastLoginAt

---

## 8.2 `UsersService`

- getMe()
- updateMe()
- admin: list/get/update users

---

## 8.3 `ProjectsService`

- CRUD operations on projects
- membership validation

---

## 8.4 `TasksService`

- createTask()
- updateTask()
- assignTask()
- updateStatus()
- listTasks() (cached)
- emit internal events

---

## 8.5 `CommentsService`

- addComment()
- listComments()
- deleteComment()

---

## 8.6 `AccessControlService`

Central permission logic for:

- project access
- task access
- manager/admin checks

---

## 8.7 `RealtimeCacheService`

Responsible for:

- Redis get/set helpers
- List cache coordination
- Invalidation logic

---

## 8.8 `RealtimeEventsService`

Responsible for:

- Standardized event payloads
- Broadcasting via Redis pub/sub
- Integration with Gateways

---

# 9) Cache + Real-Time Learning Goals

For the junior developer this project must demonstrate:

- How to design cache keys
- TTL decisions
- Cache invalidation
- Redis pub/sub usage
- WebSocket authentication
- Horizontal scaling considerations

---

# 10) Validation, Errors, and Security

---

## Validation

- All input via DTOs + class-validator
- Enums checked
- Arrays deduplicated
- IDs verified as MongoDB ObjectIds

---

## Errors

- `400` invalid input
- `401` not authenticated
- `403` permission denied
- `404` entity not found
- `409` duplicates

---

## Security

- bcrypt/argon2 password hashing
- Never expose hashes
- Basic throttling on auth
- Opaque cursors

---

# 11) Testing and Definition of Done

---

## Unit Tests

- Auth logic
- Access control
- Redis caching get/set/del
- WebSocket connection auth

---

## E2E Tests

Required flows:

- User lifecycle
- Project/task/message flow
- Real-time event reception
- Forbidden access

---

## Deliverables

- Running NestJS application
- MongoDB schemas and indexes
- Redis cache integrated
- WebSocket Gateway operational
- Swagger docs
- Tests passing in CI

---

# 12) Optional Stretch Goals

- Advanced WebSocket events
- Unread task counters
- Global activity logs
- Search service
