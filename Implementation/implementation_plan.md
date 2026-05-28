# Bookink — Kế hoạch Kiến trúc Hệ thống Toàn diện

> **Dự án học thuật** xây dựng nền tảng blog cộng đồng lấy cảm hứng từ Spiderum,  
> với kiến trúc đầy đủ: **React · Spring Boot MVC · PostgreSQL · Redis · Kafka · NiFi · Docker**

---

## 1. Tổng quan Kiến trúc Hệ thống

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                            │
│              ReactJS SPA (Vite) — i18n (EN/VI)                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP/REST (Axios)
┌──────────────────────────▼──────────────────────────────────────┐
│                    BACKEND LAYER (Spring Boot MVC)              │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────────┐   │
│  │Auth /JWT │  │Post API  │  │Comment API│  │User/Feed API │   │
│  └──────────┘  └──────────┘  └───────────┘  └──────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Service Layer                          │   │
│  │   PostService · CommentService · VoteService · ...        │   │
│  └─────────┬───────────────────┬────────────────────────────┘   │
│            │                   │                                  │
│  ┌─────────▼────┐   ┌──────────▼──────────────────────────────┐ │
│  │  Redis Cache │   │  Kafka Producer (Events)                 │ │
│  │  (Cache-Aside│   │  post.created · post.viewed              │ │
│  │   Pattern)   │   │  comment.created · vote.cast             │ │
│  └─────────┬────┘   └──────────┬──────────────────────────────┘ │
└────────────┼──────────────────┼─────────────────────────────────┘
             │                  │
┌────────────▼──────┐  ┌────────▼──────────────────────────────────┐
│   PostgreSQL      │  │         Apache Kafka (Topics)               │
│   (Main DB)       │  │  bookink.post.events                        │
│   JPA + Hibernate │  │  bookink.user.activity                      │
└───────────────────┘  │  bookink.notifications                      │
                       └────────────┬──────────────────────────────┘
                                    │
                        ┌───────────▼──────────────────────────────┐
                        │         Apache NiFi                       │
                        │  (ETL Pipeline / Data Orchestration)      │
                        │  Kafka → Transform → PostgreSQL Analytics │
                        └──────────────────────────────────────────┘
```

---

## 2. Tech Stack Chi tiết

### Backend — Spring Boot MVC

| Thành phần | Thư viện / Công nghệ | Phiên bản |
|---|---|---|
| Framework | Spring Boot | 3.x |
| Build Tool | Maven | 3.x |
| Language | Java | 21 (LTS) |
| ORM | Spring Data JPA + Hibernate | - |
| Database | PostgreSQL | 16 |
| Security | Spring Security + JWT (jjwt) | - |
| Cache | Spring Data Redis + Lettuce | - |
| Messaging | Spring Kafka | - |
| Validation | Spring Boot Validation | - |
| API Docs | Springdoc OpenAPI (Swagger UI) | 2.x |
| Migration | Flyway | - |
| Mapping | MapStruct | - |

### Frontend — ReactJS

| Thành phần | Thư viện | Phiên bản |
|---|---|---|
| Framework | React | 18 |
| Build Tool | Vite | 5.x |
| Routing | React Router DOM | 6.x |
| State | Zustand | - |
| HTTP | Axios + React Query | - |
| i18n | react-i18next + i18next | - |
| Rich Text Editor | TipTap | 2.x |
| Styling | Tailwind CSS + shadcn/ui | - |
| Form | React Hook Form + Zod | - |
| Date | date-fns | - |
| Icons | Lucide React | - |

### Infrastructure

| Thành phần | Công nghệ | Port |
|---|---|---|
| Database | PostgreSQL 16 | 5432 |
| Cache | Redis 7 | 6379 |
| Message Broker | Apache Kafka + Zookeeper | 9092 |
| Data Pipeline | Apache NiFi | 8443 |
| Kafka UI | Kafdrop | 9000 |
| Backend API | Spring Boot | 8080 |
| Frontend | Vite Dev Server | 5173 |

---

## 3. Database Schema (ERD)

### Core Tables

```sql
-- USERS
users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username    VARCHAR(50) UNIQUE NOT NULL,
  email       VARCHAR(255) UNIQUE NOT NULL,
  password    VARCHAR(255) NOT NULL,         -- BCrypt hashed
  full_name   VARCHAR(100),
  avatar_url  VARCHAR(500),
  bio         TEXT,
  role        VARCHAR(20) DEFAULT 'USER',    -- USER | ADMIN | MODERATOR
  is_active   BOOLEAN DEFAULT TRUE,
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
)

-- POSTS
posts (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title         VARCHAR(500) NOT NULL,
  slug          VARCHAR(500) UNIQUE NOT NULL,
  content       TEXT NOT NULL,              -- HTML từ TipTap
  excerpt       VARCHAR(1000),
  cover_image   VARCHAR(500),
  status        VARCHAR(20) DEFAULT 'DRAFT', -- DRAFT | PUBLISHED | ARCHIVED
  view_count    INTEGER DEFAULT 0,
  reading_time  INTEGER,                    -- phút
  author_id     UUID REFERENCES users(id),
  series_id     UUID REFERENCES series(id),
  created_at    TIMESTAMP DEFAULT NOW(),
  updated_at    TIMESTAMP DEFAULT NOW(),
  published_at  TIMESTAMP
)

-- TAGS
tags (
  id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name  VARCHAR(100) UNIQUE NOT NULL,
  slug  VARCHAR(100) UNIQUE NOT NULL,
  color VARCHAR(7)
)

-- POST_TAGS (many-to-many)
post_tags (
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  UUID REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
)

-- COMMENTS
comments (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content    TEXT NOT NULL,
  post_id    UUID REFERENCES posts(id) ON DELETE CASCADE,
  author_id  UUID REFERENCES users(id),
  parent_id  UUID REFERENCES comments(id),  -- NULL = top-level
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
)

-- VOTES
votes (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vote_type  VARCHAR(10) NOT NULL,           -- UPVOTE | DOWNVOTE
  user_id    UUID REFERENCES users(id),
  post_id    UUID REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE (user_id, post_id)
)

-- BOOKMARKS
bookmarks (
  user_id    UUID REFERENCES users(id),
  post_id    UUID REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (user_id, post_id)
)

-- FOLLOWS
follows (
  follower_id  UUID REFERENCES users(id),
  following_id UUID REFERENCES users(id),
  created_at   TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (follower_id, following_id)
)

-- SERIES
series (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title       VARCHAR(500) NOT NULL,
  description TEXT,
  author_id   UUID REFERENCES users(id),
  created_at  TIMESTAMP DEFAULT NOW()
)

-- NOTIFICATIONS
notifications (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recipient_id UUID REFERENCES users(id),
  actor_id     UUID REFERENCES users(id),
  type         VARCHAR(50) NOT NULL,    -- NEW_COMMENT | UPVOTE | NEW_FOLLOWER | etc.
  entity_id    UUID,
  entity_type  VARCHAR(50),
  is_read      BOOLEAN DEFAULT FALSE,
  created_at   TIMESTAMP DEFAULT NOW()
)

-- ANALYTICS (populated by NiFi)
post_analytics (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id    UUID REFERENCES posts(id),
  date       DATE NOT NULL,
  views      INTEGER DEFAULT 0,
  upvotes    INTEGER DEFAULT 0,
  comments   INTEGER DEFAULT 0,
  UNIQUE (post_id, date)
)
```

---

## 4. Kiến trúc Backend — Spring Boot MVC

### Cấu trúc thư mục

```
bookink-backend/
├── src/main/java/com/bookink/
│   ├── BookinkApplication.java
│   │
│   ├── controller/                   ← C trong MVC (REST Controllers)
│   │   ├── AuthController.java       POST /api/auth/register, /login, /logout
│   │   ├── PostController.java       GET/POST/PUT/DELETE /api/posts
│   │   ├── CommentController.java    GET/POST /api/posts/{id}/comments
│   │   ├── UserController.java       GET/PUT /api/users/{username}
│   │   ├── VoteController.java       POST /api/posts/{id}/vote
│   │   ├── BookmarkController.java   POST/DELETE /api/posts/{id}/bookmark
│   │   ├── TagController.java        GET /api/tags
│   │   ├── FeedController.java       GET /api/feed
│   │   ├── SearchController.java     GET /api/search
│   │   ├── NotificationController.java
│   │   └── AnalyticsController.java
│   │
│   ├── service/                      ← M trong MVC (Business Logic)
│   │   ├── AuthService.java
│   │   ├── PostService.java
│   │   ├── CommentService.java
│   │   ├── UserService.java
│   │   ├── VoteService.java
│   │   ├── FeedService.java
│   │   ├── SearchService.java
│   │   ├── NotificationService.java
│   │   └── AnalyticsService.java
│   │
│   ├── repository/                   ← Data Access Layer (Spring Data JPA)
│   │   ├── UserRepository.java
│   │   ├── PostRepository.java
│   │   ├── CommentRepository.java
│   │   ├── VoteRepository.java
│   │   ├── TagRepository.java
│   │   └── NotificationRepository.java
│   │
│   ├── model/                        ← Entities (JPA)
│   │   ├── User.java
│   │   ├── Post.java
│   │   ├── Comment.java
│   │   ├── Vote.java
│   │   ├── Tag.java
│   │   ├── Bookmark.java
│   │   ├── Follow.java
│   │   └── Notification.java
│   │
│   ├── dto/                          ← Data Transfer Objects
│   │   ├── request/
│   │   │   ├── RegisterRequest.java
│   │   │   ├── LoginRequest.java
│   │   │   ├── CreatePostRequest.java
│   │   │   └── CreateCommentRequest.java
│   │   └── response/
│   │       ├── AuthResponse.java
│   │       ├── PostResponse.java
│   │       ├── UserResponse.java
│   │       └── PageResponse.java
│   │
│   ├── kafka/
│   │   ├── producer/
│   │   │   ├── PostEventProducer.java
│   │   │   └── NotificationEventProducer.java
│   │   ├── consumer/
│   │   │   ├── AnalyticsConsumer.java
│   │   │   └── NotificationConsumer.java
│   │   └── event/
│   │       ├── PostViewedEvent.java
│   │       ├── PostCreatedEvent.java
│   │       └── VoteCastEvent.java
│   │
│   ├── cache/
│   │   ├── RedisConfig.java
│   │   ├── FeedCacheService.java     -- Cache trending posts
│   │   └── ViewCountBuffer.java      -- Buffer view counts
│   │
│   ├── security/
│   │   ├── SecurityConfig.java
│   │   ├── JwtAuthFilter.java
│   │   ├── JwtTokenProvider.java
│   │   └── UserDetailsServiceImpl.java
│   │
│   ├── config/
│   │   ├── KafkaConfig.java
│   │   ├── CorsConfig.java
│   │   └── SwaggerConfig.java
│   │
│   └── exception/
│       ├── GlobalExceptionHandler.java
│       ├── ResourceNotFoundException.java
│       └── UnauthorizedException.java
│
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   └── db/migration/                  ← Flyway migrations
│       ├── V1__create_users.sql
│       ├── V2__create_posts.sql
│       └── V3__create_interactions.sql
│
└── pom.xml
```

---

## 5. Cấu trúc Frontend — ReactJS

```
bookink-frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   │
│   ├── pages/
│   │   ├── Home/              -- Feed bài viết (Trending | Mới nhất | Đang theo dõi)
│   │   ├── Auth/
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   ├── Post/
│   │   │   ├── PostDetail.tsx    -- Chi tiết bài viết
│   │   │   ├── PostEditor.tsx    -- Tạo/Sửa bài (TipTap editor)
│   │   │   └── PostList.tsx
│   │   ├── Profile/
│   │   │   └── UserProfile.tsx   -- Profile + bài viết của user
│   │   ├── Tag/
│   │   │   └── TagPage.tsx       -- Tất cả bài theo tag
│   │   ├── Search/
│   │   │   └── SearchResults.tsx
│   │   ├── Dashboard/
│   │   │   ├── MyPosts.tsx
│   │   │   ├── Analytics.tsx     -- Biểu đồ thống kê
│   │   │   └── Bookmarks.tsx
│   │   └── Notifications/
│   │       └── NotificationsPage.tsx
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Navbar.tsx        -- Header với search + lang toggle
│   │   │   ├── Sidebar.tsx       -- Trending tags, Suggested users
│   │   │   └── Footer.tsx
│   │   ├── post/
│   │   │   ├── PostCard.tsx      -- Card bài viết trong feed
│   │   │   ├── PostVote.tsx      -- Upvote/Downvote widget
│   │   │   └── PostStats.tsx
│   │   ├── comment/
│   │   │   ├── CommentList.tsx
│   │   │   ├── CommentItem.tsx
│   │   │   └── CommentForm.tsx
│   │   ├── editor/
│   │   │   ├── TipTapEditor.tsx
│   │   │   └── EditorToolbar.tsx
│   │   └── ui/                   -- shadcn/ui components
│   │
│   ├── store/                    -- Zustand stores
│   │   ├── authStore.ts
│   │   ├── postStore.ts
│   │   └── uiStore.ts
│   │
│   ├── services/                 -- Axios API calls
│   │   ├── api.ts                -- Axios instance + interceptors
│   │   ├── authService.ts
│   │   ├── postService.ts
│   │   ├── commentService.ts
│   │   └── userService.ts
│   │
│   ├── hooks/                    -- Custom React Query hooks
│   │   ├── usePosts.ts
│   │   ├── usePost.ts
│   │   └── useAuth.ts
│   │
│   ├── i18n/
│   │   ├── index.ts              -- i18next config
│   │   └── locales/
│   │       ├── vi/
│   │       │   ├── common.json
│   │       │   ├── post.json
│   │       │   └── auth.json
│   │       └── en/
│   │           ├── common.json
│   │           ├── post.json
│   │           └── auth.json
│   │
│   └── utils/
│       ├── slugify.ts
│       ├── readingTime.ts
│       └── formatDate.ts
│
├── index.html
├── vite.config.ts
├── tailwind.config.ts
└── package.json
```

---

## 6. Redis — Chiến lược Cache

```
┌─────────────────────────────────────────────────────┐
│                    Redis Keys                         │
├─────────────────────────┬───────────────────────────┤
│ feed:trending           │ List<PostId> — Top 20 bài  │
│                         │ trending (TTL: 10 phút)    │
├─────────────────────────┼───────────────────────────┤
│ post:{id}               │ PostResponse JSON          │
│                         │ (TTL: 30 phút)             │
├─────────────────────────┼───────────────────────────┤
│ post:{id}:views         │ Counter — View count buffer│
│                         │ Flush to DB every 5 phút   │
├─────────────────────────┼───────────────────────────┤
│ post:{id}:votes         │ Hash{upvotes, downvotes}   │
│                         │ (TTL: 15 phút)             │
├─────────────────────────┼───────────────────────────┤
│ user:{id}:profile       │ UserResponse JSON          │
│                         │ (TTL: 60 phút)             │
├─────────────────────────┼───────────────────────────┤
│ ratelimit:{userId}:vote │ Counter (TTL: 1 phút)      │
│                         │ Chống spam vote            │
└─────────────────────────┴───────────────────────────┘
```

**Pattern sử dụng**: Cache-Aside (Lazy Loading)
- Check Redis trước → Cache hit: trả về ngay
- Cache miss: query PostgreSQL → lưu vào Redis → trả về

---

## 7. Kafka — Luồng Sự kiện

### Topics

| Topic | Producer | Consumer | Dữ liệu |
|---|---|---|---|
| `bookink.post.viewed` | PostService | AnalyticsConsumer, NiFi | `{postId, userId, timestamp, ip}` |
| `bookink.post.created` | PostService | NotificationConsumer, NiFi | `{postId, authorId, title, tags}` |
| `bookink.vote.cast` | VoteService | AnalyticsConsumer | `{postId, userId, voteType}` |
| `bookink.comment.created` | CommentService | NotificationConsumer | `{commentId, postId, authorId}` |
| `bookink.user.followed` | UserService | NotificationConsumer | `{followerId, followingId}` |
| `bookink.notifications` | NotificationConsumer | Frontend (SSE) | `{recipientId, type, message}` |

### Luồng Notification

```
User upvotes a post
     │
     ▼
VoteService.castVote()
     │
     ├── Update DB (votes table)
     ├── Invalidate Redis cache
     └── Kafka.produce("bookink.vote.cast", event)
                    │
                    ▼
           NotificationConsumer
                    │
                    ▼
           Save to notifications table
                    │
                    ▼
           Kafka.produce("bookink.notifications", {recipientId: post.authorId})
                    │
                    ▼
           Frontend SSE stream → Notification badge update
```

---

## 8. Apache NiFi — Data Pipeline

### Pipeline 1: Analytics ETL

```
[Kafka Consumer]     [Transform JSON]    [Insert to DB]
bookink.post.viewed ──► ExtractFields ──► ExecuteSQL ──► post_analytics
bookink.vote.cast   ──► MergeContent ──►    (daily aggregation)
```

### Pipeline 2: Content Export/Import

```
[DB Query] ──► [Convert to JSON/CSV] ──► [FileSystem / S3]
 posts table        Transform                  Export
```

### NiFi Processors sử dụng

| Processor | Mục đích |
|---|---|
| `ConsumeKafka` | Đọc events từ Kafka topics |
| `EvaluateJsonPath` | Trích xuất fields từ JSON event |
| `UpdateAttribute` | Thêm metadata (timestamp, date) |
| `MergeContent` | Gom nhóm events để batch insert |
| `ExecuteSQL` | Aggregation và insert vào `post_analytics` |
| `PutDatabaseRecord` | Ghi dữ liệu có cấu trúc vào PostgreSQL |
| `LogAttribute` | Debug và monitoring |

---

## 9. REST API Design

### Authentication
```
POST /api/auth/register     -- Đăng ký
POST /api/auth/login        -- Đăng nhập → JWT token
POST /api/auth/logout       -- Invalidate token
POST /api/auth/refresh      -- Refresh JWT
```

### Posts
```
GET    /api/posts?page=0&size=10&sort=trending   -- Feed
GET    /api/posts/{slug}                          -- Chi tiết bài
POST   /api/posts                                 -- Tạo bài [AUTH]
PUT    /api/posts/{id}                            -- Sửa bài [AUTH + AUTHOR]
DELETE /api/posts/{id}                            -- Xóa bài [AUTH + AUTHOR]
POST   /api/posts/{id}/publish                    -- Xuất bản [AUTH]
```

### Interactions
```
POST   /api/posts/{id}/vote          -- Vote {type: "UPVOTE"|"DOWNVOTE"} [AUTH]
DELETE /api/posts/{id}/vote          -- Bỏ vote [AUTH]
POST   /api/posts/{id}/bookmark      -- Lưu bài [AUTH]
DELETE /api/posts/{id}/bookmark      -- Bỏ lưu [AUTH]
GET    /api/posts/{id}/comments      -- Lấy comments (phân trang)
POST   /api/posts/{id}/comments      -- Thêm comment [AUTH]
DELETE /api/comments/{id}            -- Xóa comment [AUTH + AUTHOR]
```

### Users
```
GET  /api/users/{username}           -- Profile
GET  /api/users/{username}/posts     -- Bài viết của user
POST /api/users/{username}/follow    -- Follow [AUTH]
DELETE /api/users/{username}/follow  -- Unfollow [AUTH]
GET  /api/users/me                   -- My profile [AUTH]
PUT  /api/users/me                   -- Update profile [AUTH]
```

### Search & Tags
```
GET /api/search?q=keyword&type=posts|users
GET /api/tags
GET /api/tags/{slug}/posts
```

### Analytics (Dashboard)
```
GET /api/analytics/posts/{id}       -- Stats bài viết [AUTH + AUTHOR]
GET /api/analytics/overview         -- Tổng quan author [AUTH]
```

---

## 10. Docker Compose — Toàn bộ Infrastructure

```yaml
# docker-compose.yml (tóm tắt)
services:
  postgres:        # PostgreSQL 16    — port 5432
  redis:           # Redis 7          — port 6379
  zookeeper:       # Zookeeper 3.8    — port 2181
  kafka:           # Kafka 3.x        — port 9092
  kafdrop:         # Kafka UI         — port 9000
  nifi:            # Apache NiFi      — port 8443
  backend:         # Spring Boot      — port 8080
  frontend:        # Vite (build)     — port 5173
```

---

## 11. Tính năng Đầy đủ (Full Feature List)

### ✅ Authentication & Users
- [ ] Đăng ký với email/password (BCrypt hash)
- [ ] Đăng nhập → JWT Access Token + Refresh Token
- [ ] Token blacklist trong Redis (logout)
- [ ] Xem & chỉnh sửa profile (avatar, bio, social links)
- [ ] Follow / Unfollow người dùng

### ✅ Posts
- [ ] Tạo/Sửa/Xóa bài với Rich Text Editor (TipTap)
  - Bold, Italic, Heading, Blockquote, Code block, Image, Link
  - Table of Contents tự động
- [ ] Upload ảnh bìa (cover image)
- [ ] Gán Tags (tối đa 5 tags)
- [ ] Draft / Published / Archived
- [ ] SEO-friendly slug URL
- [ ] Ước tính thời gian đọc (reading time)
- [ ] View count (buffered qua Redis → flush DB)

### ✅ Feed & Discovery
- [ ] Trang chủ: Trending | Mới nhất | Đang theo dõi
- [ ] Phân trang (Pagination + Infinite scroll)
- [ ] Trang tag (xem bài theo tag)
- [ ] Tìm kiếm toàn văn (PostgreSQL FTS)

### ✅ Interactions
- [ ] Upvote / Downvote bài viết
- [ ] Bình luận 2 cấp (top-level + reply)
- [ ] Bookmark / Lưu bài viết
- [ ] Chia sẻ (copy link)

### ✅ Notifications
- [ ] Thông báo: upvote, comment, follow mới
- [ ] Real-time via Server-Sent Events (SSE) hoặc polling
- [ ] Đánh dấu đã đọc

### ✅ Dashboard Tác giả
- [ ] Thống kê: views, upvotes, bookmarks theo ngày
- [ ] Biểu đồ (Chart.js / Recharts)
- [ ] Quản lý bài viết (draft/published)

### ✅ i18n
- [ ] Chuyển ngôn ngữ VI ↔ EN không cần reload
- [ ] Tất cả text UI dịch ra 2 ngôn ngữ
- [ ] Lưu lựa chọn ngôn ngữ vào localStorage

### ✅ Admin (Bonus)
- [ ] Dashboard admin: quản lý users, posts, tags
- [ ] Ẩn/Xóa nội dung vi phạm

---

## 12. Kế hoạch Thực hiện (Timeline)

| Giai đoạn | Công việc | Ngày ước tính |
|---|---|---|
| **Phase 0** | Setup project, Docker Compose, DB migration | 2 ngày |
| **Phase 1** | Backend Auth + User CRUD + Spring Security + JWT | 2 ngày |
| **Phase 2** | Backend Post CRUD (Controller → Service → Repo) | 2 ngày |
| **Phase 3** | Backend: Comments, Votes, Bookmarks, Follow | 1.5 ngày |
| **Phase 4** | Redis integration (cache, view buffer, rate limit) | 1 ngày |
| **Phase 5** | Kafka integration (producers, consumers, events) | 1.5 ngày |
| **Phase 6** | NiFi pipeline (ETL analytics) | 1 ngày |
| **Phase 7** | Frontend setup + i18n + Layout + Auth pages | 2 ngày |
| **Phase 8** | Frontend: Home feed + Post detail + Editor | 2.5 ngày |
| **Phase 9** | Frontend: Profile + Follow + Notifications | 1.5 ngày |
| **Phase 10** | Frontend: Dashboard + Analytics charts | 1 ngày |
| **Phase 11** | Frontend: Search + Tags + Bookmarks | 1 ngày |
| **Phase 12** | Testing, Swagger docs, code cleanup | 1 ngày |
| **Tổng** | | **~20 ngày** |

---

## 13. Verification Plan

### Backend
- `mvn test` — Unit tests (JUnit 5 + Mockito)
- `mvn spring-boot:run` + Swagger UI tại `localhost:8080/swagger-ui.html`
- Test API endpoints với Postman/HTTPie

### Frontend
- `npm run dev` — Dev server tại `localhost:5173`
- Kiểm tra i18n switch EN/VI
- Kiểm tra editor TipTap
- Kiểm tra responsive (mobile/tablet/desktop)

### Infrastructure
- `docker-compose up` — Tất cả services khởi động
- Kafdrop UI `localhost:9000` — Xem Kafka topics và messages
- NiFi UI `localhost:8443/nifi` — Xem data pipelines
- Redis CLI: `redis-cli KEYS "*"` — Kiểm tra cache

### E2E Flow Test
1. Register → Login → JWT nhận được ✓
2. Tạo bài viết → Publish → Xuất hiện trong feed ✓
3. Upvote bài → Kafka event gửi → Analytics cập nhật ✓
4. Comment → Notification gửi tới tác giả ✓
5. Feed trending caching qua Redis ✓
6. NiFi pipeline ETL → `post_analytics` có data ✓

---

> [!NOTE]
> **Câu hỏi còn lại**: Bạn đã có thiết kế giao diện rồi — bạn có thể chia sẻ file thiết kế (Figma, ảnh, PDF...) không?  
> Điều này giúp tôi implement frontend chính xác theo design của bạn ngay từ đầu.
