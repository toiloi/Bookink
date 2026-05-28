# Bookink — Task Breakdown (Step by Step)

> Hướng dẫn chi tiết từng bước xây dựng hệ thống.  
> Không cần đọc theo thứ tự tuyến tính — mỗi Phase là một khối độc lập.  
> Trạng thái: `[ ]` Chưa làm · `[/]` Đang làm · `[x]` Xong

---

## PHASE 0 — Setup Dự án & Infrastructure

### 0.1 Khởi tạo Repository & Cấu trúc thư mục

- [ ] Tạo cấu trúc thư mục gốc trong repo:
  ```
  Bookink/
  ├── bookink-backend/     ← Spring Boot
  ├── bookink-frontend/    ← ReactJS
  ├── docker/              ← Dockerfile riêng cho từng service
  ├── docker-compose.yml   ← Toàn bộ infrastructure
  ├── .env.example         ← Mẫu biến môi trường
  └── README.md
  ```
- [ ] Tạo file `.gitignore` bao gồm: `target/`, `node_modules/`, `.env`, `*.class`, `*.jar`
- [ ] Tạo file `.env.example` với tất cả các biến môi trường cần thiết (không chứa giá trị thật)

---

### 0.2 Docker Compose — Toàn bộ Infrastructure

> **Mục tiêu**: Chạy `docker-compose up -d` là có đủ mọi thứ

- [ ] **Tạo `docker-compose.yml`** với các services sau:

  **PostgreSQL 16**
  - Image: `postgres:16-alpine`
  - Port: `5432:5432`
  - Volume: `postgres_data:/var/lib/postgresql/data`
  - Env: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`

  **Redis 7**
  - Image: `redis:7-alpine`
  - Port: `6379:6379`
  - Command: `redis-server --requirepass <password>`

  **Zookeeper**
  - Image: `confluentinc/cp-zookeeper:7.5.0`
  - Port: `2181:2181`
  - Env: `ZOOKEEPER_CLIENT_PORT=2181`

  **Kafka**
  - Image: `confluentinc/cp-kafka:7.5.0`
  - Port: `9092:9092`
  - Depends on: `zookeeper`
  - Env: `KAFKA_BROKER_ID`, `KAFKA_ZOOKEEPER_CONNECT`, `KAFKA_ADVERTISED_LISTENERS`

  **Kafdrop** (Kafka UI)
  - Image: `obsidiandynamics/kafdrop:latest`
  - Port: `9000:9000`
  - Env: `KAFKA_BROKERCONNECT=kafka:9092`

  **Apache NiFi**
  - Image: `apache/nifi:1.23.2`
  - Port: `8443:8443`
  - Volume: `nifi_data:/opt/nifi/nifi-current/conf`
  - Env: `SINGLE_USER_CREDENTIALS_USERNAME`, `SINGLE_USER_CREDENTIALS_PASSWORD`

- [ ] Kiểm tra: `docker-compose up -d` → tất cả containers `healthy`
- [ ] Truy cập Kafdrop tại `http://localhost:9000` — thấy giao diện
- [ ] Truy cập NiFi tại `https://localhost:8443/nifi` — login thành công

---

### 0.3 Khởi tạo Spring Boot Project

- [ ] Vào [start.spring.io](https://start.spring.io) và tạo project với:
  - **Project**: Maven
  - **Language**: Java 21
  - **Spring Boot**: 3.3.x
  - **Group**: `com.bookink`
  - **Artifact**: `bookink-backend`
  - **Dependencies**:
    - Spring Web
    - Spring Data JPA
    - Spring Security
    - Spring Data Redis (Lettuce)
    - Spring for Apache Kafka
    - PostgreSQL Driver
    - Flyway Migration
    - Lombok
    - Validation
    - Springdoc OpenAPI UI
- [ ] Download và giải nén vào `bookink-backend/`
- [ ] Import vào IDE (IntelliJ IDEA khuyến nghị)
- [ ] Thêm dependency `jjwt` (JWT) vào `pom.xml`:
  ```xml
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
  </dependency>
  ```
- [ ] Thêm dependency `mapstruct` vào `pom.xml` (DTO mapping)
- [ ] Chạy `mvn clean install` — build thành công (dù chưa có code)

---

### 0.4 Khởi tạo ReactJS Frontend

- [ ] Chạy lệnh khởi tạo Vite + React + TypeScript:
  ```bash
  npm create vite@latest bookink-frontend -- --template react-ts
  ```
- [ ] Di chuyển vào `bookink-frontend/`, cài dependencies:
  ```bash
  npm install
  npm install react-router-dom axios @tanstack/react-query zustand
  npm install react-i18next i18next i18next-browser-languagedetector
  npm install @tiptap/react @tiptap/starter-kit @tiptap/extension-image
  npm install react-hook-form zod @hookform/resolvers
  npm install date-fns lucide-react
  npm install recharts           # biểu đồ analytics
  ```
- [ ] Cài Tailwind CSS:
  ```bash
  npm install -D tailwindcss postcss autoprefixer
  npx tailwindcss init -p
  ```
- [ ] Cài shadcn/ui:
  ```bash
  npx shadcn-ui@latest init
  ```
- [ ] Chạy `npm run dev` — thấy trang Vite default tại `localhost:5173`

---

## PHASE 1 — Backend: Database Schema & Cấu hình

### 1.1 Cấu hình `application.yml`

- [ ] Tạo `src/main/resources/application.yml`:
  ```yaml
  spring:
    datasource:
      url: jdbc:postgresql://localhost:5432/bookink
      username: ${DB_USER}
      password: ${DB_PASSWORD}
    jpa:
      hibernate:
        ddl-auto: validate        # Flyway quản lý schema
      show-sql: false
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
    kafka:
      bootstrap-servers: localhost:9092

  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400000         # 24h (ms)
    refresh-expiration: 604800000 # 7 ngày

  springdoc:
    api-docs:
      path: /api-docs
    swagger-ui:
      path: /swagger-ui.html
  ```
- [ ] Tạo `application-dev.yml` override cho môi trường dev (log SQL)

---

### 1.2 Flyway Database Migrations

> Tất cả schema tạo qua Flyway migration, **không dùng** `ddl-auto: create`

- [ ] Tạo thư mục `src/main/resources/db/migration/`
- [ ] **V1__create_users.sql** — Bảng `users` với UUID, email unique, role enum
- [ ] **V2__create_posts.sql** — Bảng `posts`, `tags`, `post_tags` (many-to-many)
- [ ] **V3__create_interactions.sql** — Bảng `comments`, `votes`, `bookmarks`, `follows`
- [ ] **V4__create_notifications.sql** — Bảng `notifications`
- [ ] **V5__create_analytics.sql** — Bảng `post_analytics` (date, views, upvotes, comments)
- [ ] **V6__create_series.sql** — Bảng `series`
- [ ] **V7__seed_tags.sql** — Insert các tags mẫu (Công nghệ, Khoa học, Tâm lý, ...)
- [ ] Chạy Spring Boot → Flyway tự chạy migrations → kiểm tra DB bằng DBeaver/TablePlus

---

### 1.3 JPA Entities

> Mỗi entity tương ứng 1 bảng, dùng Lombok `@Data`, `@Builder`, `@NoArgsConstructor`

- [ ] **`User.java`** — fields: id, username, email, password, fullName, avatarUrl, bio, role, isActive, createdAt
- [ ] **`Post.java`** — fields: id, title, slug, content (Text), excerpt, coverImage, status (Enum), viewCount, readingTime, author (ManyToOne User), tags (ManyToMany Tag), createdAt, publishedAt
- [ ] **`Tag.java`** — fields: id, name, slug, color
- [ ] **`Comment.java`** — fields: id, content, post (ManyToOne), author (ManyToOne), parent (ManyToOne Comment tự tham chiếu), replies (OneToMany)
- [ ] **`Vote.java`** — fields: id, voteType (Enum: UPVOTE/DOWNVOTE), user, post — `@UniqueConstraint(userId, postId)`
- [ ] **`Bookmark.java`** — Composite key (userId + postId)
- [ ] **`Follow.java`** — Composite key (followerId + followingId)
- [ ] **`Notification.java`** — fields: id, recipient, actor, type (Enum), entityId, entityType, isRead, createdAt
- [ ] **`PostAnalytics.java`** — fields: postId, date, views, upvotes, comments

---

### 1.4 Repository Layer (Spring Data JPA)

- [ ] **`UserRepository`** — `findByEmail()`, `findByUsername()`, `existsByEmail()`
- [ ] **`PostRepository`** — `findBySlug()`, `findByAuthorId(Pageable)`, custom query trending feed (sort by votes + views)
- [ ] **`CommentRepository`** — `findByPostIdAndParentIsNull(Pageable)` (top-level comments)
- [ ] **`VoteRepository`** — `findByUserIdAndPostId()`, `countByPostIdAndVoteType()`
- [ ] **`TagRepository`** — `findBySlug()`
- [ ] **`NotificationRepository`** — `findByRecipientIdOrderByCreatedAtDesc(Pageable)`, `countByRecipientIdAndIsReadFalse()`
- [ ] **`PostAnalyticsRepository`** — `findByPostIdAndDate()`, custom query tổng hợp

---

## PHASE 2 — Backend: Authentication & Security

### 2.1 JWT Token Provider

- [ ] Tạo class **`JwtTokenProvider`** với các methods:
  - `generateAccessToken(UserDetails)` → trả về JWT string (24h)
  - `generateRefreshToken(UserDetails)` → trả về JWT string (7 ngày)
  - `validateToken(String token)` → boolean
  - `getUsernameFromToken(String token)` → String
- [ ] JWT secret lưu trong biến môi trường `JWT_SECRET` (≥ 256-bit)
- [ ] Access token chứa claims: `sub` (username), `userId`, `role`, `iat`, `exp`

---

### 2.2 Spring Security Configuration

- [ ] Tạo **`SecurityConfig`** extends `WebSecurityConfigurerAdapter` (hoặc `SecurityFilterChain` bean):
  - Disable CSRF (REST API)
  - Session management: `STATELESS`
  - Public endpoints (no auth): `POST /api/auth/**`, `GET /api/posts/**`, `GET /api/tags/**`, `GET /api/users/**`, `GET /api/search`
  - Authenticated endpoints: tất cả `POST`, `PUT`, `DELETE`
- [ ] Tạo **`JwtAuthFilter`** extends `OncePerRequestFilter`:
  - Đọc header `Authorization: Bearer <token>`
  - Validate token → set `SecurityContext`
- [ ] Đăng ký filter trước `UsernamePasswordAuthenticationFilter`
- [ ] Tạo **`UserDetailsServiceImpl`** — load user từ DB theo email
- [ ] Password encoder: `BCryptPasswordEncoder` bean

---

### 2.3 Auth DTOs & Controller

- [ ] **DTOs**:
  - `RegisterRequest` — email, password, username, fullName (có validation: @NotBlank, @Email, @Size)
  - `LoginRequest` — email, password
  - `AuthResponse` — accessToken, refreshToken, user info
- [ ] **`AuthController`** (`/api/auth`):
  - `POST /register` → validate → hash password → save user → trả về `AuthResponse`
  - `POST /login` → authenticate → tạo JWT → trả về `AuthResponse`
  - `POST /logout` → blacklist token trong Redis (key: `blacklist:{token}`, TTL = token TTL còn lại)
  - `POST /refresh` → validate refresh token → cấp access token mới
- [ ] **`AuthService`** chứa logic business, không viết trong controller

---

## PHASE 3 — Backend: Post CRUD

### 3.1 Post Service & Controller

- [ ] **DTOs**:
  - `CreatePostRequest` — title, content, excerpt, coverImage, tagIds (list), status
  - `UpdatePostRequest` — tương tự CreatePostRequest
  - `PostResponse` — tất cả fields + authorName, authorAvatar, tagNames, upvoteCount, downvoteCount, commentCount, isBookmarked (nếu có auth), isVoted
  - `PostSummaryResponse` — dùng cho feed (không có full content)

- [ ] **`PostService`**:
  - `createPost(CreatePostRequest, userId)` → tự động generate slug từ title (slugify) → lưu DB → publish Kafka event `bookink.post.created`
  - `getPostBySlug(slug, userId?)` → check Redis cache trước → DB → cache lại → tăng view count (Redis buffer)
  - `updatePost(id, UpdatePostRequest, userId)` → check ownership → update → invalidate cache
  - `deletePost(id, userId)` → check ownership → soft delete (status = ARCHIVED) hoặc hard delete
  - `publishPost(id, userId)` → set status = PUBLISHED, publishedAt = now()
  - `getTrendingFeed(page, size)` → check Redis `feed:trending` → DB query → cache
  - `getLatestFeed(page, size)` → DB query không cache
  - `getFollowingFeed(userId, page, size)` → query bài của người đang follow
  - `getPostsByAuthor(username, page, size)`

- [ ] **`PostController`** (`/api/posts`):
  - `GET /` với query params: `?sort=trending|latest|following&page=0&size=10`
  - `GET /{slug}`
  - `POST /` [AUTH]
  - `PUT /{id}` [AUTH]
  - `DELETE /{id}` [AUTH]
  - `POST /{id}/publish` [AUTH]

---

### 3.2 Slug & Reading Time Utils

- [ ] Tạo **`SlugUtils`**: chuyển tiếng Việt có dấu → không dấu → kebab-case → thêm UUID suffix ngắn (tránh trùng)
  - Ví dụ: `"Học Spring Boot"` → `"hoc-spring-boot-a1b2"`
- [ ] Tạo **`ReadingTimeUtils`**: đếm số từ trong content HTML → chia cho 200 (từ/phút) → làm tròn lên

---

### 3.3 Tags API

- [ ] **`TagController`** (`/api/tags`):
  - `GET /` → Tất cả tags (cache Redis)
  - `GET /{slug}/posts` → Tất cả bài viết theo tag (phân trang)
- [ ] Admin có thể tạo/sửa/xóa tag: `POST /api/admin/tags`

---

## PHASE 4 — Backend: Interactions (Comment, Vote, Bookmark, Follow)

### 4.1 Comment System

- [ ] **DTOs**: `CreateCommentRequest` (content, parentId?), `CommentResponse` (id, content, author, createdAt, replies?)
- [ ] **`CommentService`**:
  - `addComment(postId, CreateCommentRequest, userId)` → lưu DB → publish Kafka event `bookink.comment.created`
  - `getRootComments(postId, page, size)` → lấy comments không có parent, kèm 3 replies đầu
  - `getReplies(commentId, page, size)` → load more replies
  - `deleteComment(commentId, userId)` → check ownership
- [ ] **`CommentController`** (`/api/posts/{postId}/comments`):
  - `GET /` — danh sách top-level comments
  - `POST /` [AUTH]
  - `GET /{commentId}/replies`
  - `DELETE /api/comments/{id}` [AUTH]

---

### 4.2 Vote System

- [ ] **`VoteService`**:
  - `castVote(postId, voteType, userId)`:
    - Nếu chưa vote → insert vote mới
    - Nếu đã vote cùng loại → xóa vote (toggle off)
    - Nếu đã vote khác loại → update vote type
    - Invalidate Redis cache cho post đó
    - Publish Kafka event `bookink.vote.cast`
  - `getVoteSummary(postId)` → check Redis → DB count
- [ ] **`VoteController`** (`/api/posts/{postId}/vote`):
  - `POST /` [AUTH] — body: `{"voteType": "UPVOTE"}`
  - `DELETE /` [AUTH]

---

### 4.3 Bookmark & Follow

- [ ] **`BookmarkService`**: `addBookmark(postId, userId)`, `removeBookmark(postId, userId)`, `getBookmarks(userId, page)`
- [ ] **`BookmarkController`** (`/api/posts/{postId}/bookmark`): `POST /`, `DELETE /`
- [ ] **`FollowService`**: `follow(targetUserId, currentUserId)`, `unfollow()`, `getFollowers(userId)`, `getFollowing(userId)` → publish Kafka event `bookink.user.followed`
- [ ] **`FollowController`** (`/api/users/{username}/follow`): `POST /`, `DELETE /`

---

### 4.4 Search

- [ ] **PostgreSQL Full-Text Search** (không cần Elasticsearch cho MVP):
  - Thêm `tsvector` column vào `posts` hoặc dùng `to_tsvector()` inline
  - Migration: `CREATE INDEX posts_fts_idx ON posts USING GIN (to_tsvector('english', title || ' ' || excerpt))`
- [ ] **`SearchController`** (`/api/search`):
  - `GET /?q=keyword&type=posts|users&page=0&size=10`
  - Tìm kiếm cả tiếng Việt (dùng `unaccent` extension của PostgreSQL)

---

## PHASE 5 — Backend: Redis Integration

### 5.1 Redis Configuration

- [ ] Tạo **`RedisConfig`** bean:
  - `RedisTemplate<String, Object>` với Jackson serializer
  - `CacheManager` với TTL mặc định 30 phút
- [ ] Tạo constants class **`RedisKeys`**:
  ```java
  String FEED_TRENDING = "feed:trending";
  String POST_DETAIL = "post:%s";          // post:{slug}
  String POST_VIEWS = "post:%s:views";     // post:{id}:views (counter)
  String USER_PROFILE = "user:%s:profile"; // user:{username}:profile
  String RATE_LIMIT_VOTE = "ratelimit:%s:vote:%s"; // user:post
  ```

---

### 5.2 Cache-Aside Pattern Implementation

- [ ] **Feed Caching** trong `PostService.getTrendingFeed()`:
  - Check `feed:trending` trong Redis
  - Hit → deserialize và trả về
  - Miss → query DB → serialize → `redisTemplate.opsForValue().set(key, value, 10, MINUTES)` → trả về
- [ ] **Post Detail Caching** trong `PostService.getPostBySlug()`:
  - Cache post response với TTL 30 phút
  - Invalidate khi post được update/delete

---

### 5.3 View Count Buffer

- [ ] **`ViewCountBuffer`** service:
  - `increment(postId)` → `redisTemplate.opsForValue().increment("post:{id}:views")`
  - Scheduled task (mỗi 5 phút): `@Scheduled(fixedRate = 300000)` → đọc tất cả keys `post:*:views` → batch update DB → reset counter về 0
- [ ] Lý do: tránh write DB mỗi lần có view → giảm tải database

---

### 5.4 Rate Limiting (Chống Spam)

- [ ] **`RateLimitService`**:
  - `isAllowed(userId, action)` → check Redis counter
  - Ví dụ vote: `ratelimit:{userId}:vote:{postId}` — nếu key tồn tại → blocked (TTL 60 giây)
  - Set key với TTL sau mỗi action thành công
- [ ] Gọi trong `VoteService` trước khi xử lý vote

---

## PHASE 6 — Backend: Kafka Integration

### 6.1 Kafka Configuration

- [ ] **`KafkaConfig`**:
  - `ProducerFactory` với serializer JSON
  - `KafkaTemplate<String, Object>` bean
  - `ConsumerFactory` với deserializer JSON
  - `ConcurrentKafkaListenerContainerFactory` với error handler

- [ ] Tạo **Kafka Topics** (tạo programmatically qua `NewTopic` beans hoặc trong docker-compose):
  - `bookink.post.events` (3 partitions, 1 replica)
  - `bookink.user.activity` (3 partitions)
  - `bookink.notifications` (3 partitions)
  - `bookink.analytics` (3 partitions)

---

### 6.2 Event Classes (Kafka Messages)

- [ ] **`PostViewedEvent`** — postId, userId (nullable), ip, userAgent, timestamp
- [ ] **`PostCreatedEvent`** — postId, authorId, title, tags[], publishedAt
- [ ] **`VoteCastEvent`** — postId, userId, voteType, timestamp
- [ ] **`CommentCreatedEvent`** — commentId, postId, postAuthorId, commentAuthorId, timestamp
- [ ] **`UserFollowedEvent`** — followerId, followingId, timestamp

---

### 6.3 Kafka Producers

- [ ] **`PostEventProducer`**:
  - `publishPostViewed(PostViewedEvent)` → `kafkaTemplate.send("bookink.analytics", event)`
  - `publishPostCreated(PostCreatedEvent)` → `kafkaTemplate.send("bookink.post.events", event)`
- [ ] **`InteractionEventProducer`**:
  - `publishVoteCast(VoteCastEvent)` → topic `bookink.analytics`
  - `publishCommentCreated(CommentCreatedEvent)` → topic `bookink.notifications`
  - `publishUserFollowed(UserFollowedEvent)` → topic `bookink.notifications`

---

### 6.4 Kafka Consumers

- [ ] **`NotificationConsumer`** — lắng nghe `bookink.notifications`:
  - Nhận event → tạo `Notification` record trong DB
  - Gọi `NotificationService.create(notification)`
  - (Optional) Push thông báo realtime qua SSE

- [ ] **`AnalyticsConsumer`** — lắng nghe `bookink.analytics`:
  - Nhận `PostViewedEvent` → upsert `post_analytics` (tăng views theo ngày)
  - Nhận `VoteCastEvent` → upsert `post_analytics` (tăng upvotes theo ngày)

---

### 6.5 Notification API

- [ ] **`NotificationController`** (`/api/notifications`) [AUTH]:
  - `GET /` — Danh sách thông báo (phân trang)
  - `GET /unread-count` — Đếm chưa đọc
  - `PUT /{id}/read` — Đánh dấu đã đọc
  - `PUT /read-all` — Đánh dấu tất cả đã đọc

---

## PHASE 7 — Apache NiFi: Data Pipeline

### 7.1 Cài đặt & Login NiFi

- [ ] NiFi đã chạy qua Docker Compose
- [ ] Truy cập `https://localhost:8443/nifi`
- [ ] Login với credentials đã set trong docker-compose
- [ ] Tạo Process Group mới: `Bookink Analytics Pipeline`

---

### 7.2 Pipeline 1: Kafka → Analytics DB

> **Mục đích**: Đọc events từ Kafka, transform, aggregate và lưu vào `post_analytics`

- [ ] **Processor 1**: `ConsumeKafka_2_6`
  - Topic: `bookink.analytics`
  - Group ID: `nifi-analytics-consumer`
  - Output: FlowFile với JSON content

- [ ] **Processor 2**: `EvaluateJsonPath`
  - Trích xuất: `postId`, `eventType`, `timestamp` → lưu vào Attributes

- [ ] **Processor 3**: `RouteOnAttribute`
  - Route theo `eventType`: `POST_VIEWED` → đường A, `VOTE_CAST` → đường B

- [ ] **Processor 4**: `UpdateAttribute`
  - Thêm attribute: `date` = ngày từ timestamp (format: `yyyy-MM-dd`)

- [ ] **Processor 5**: `ExecuteSQL` hoặc `PutDatabaseRecord`
  - Kết nối PostgreSQL (thêm JDBC driver PostgreSQL vào NiFi)
  - SQL: `INSERT INTO post_analytics (post_id, date, views, upvotes) VALUES (?, ?, 1, 0) ON CONFLICT (post_id, date) DO UPDATE SET views = views + 1`
  - Tương tự cho upvotes

- [ ] **Test**: Upvote 1 bài → xem Kafka message trong Kafdrop → xem data trong `post_analytics` table

---

### 7.3 Pipeline 2 (Bonus): Export bài viết ra JSON

- [ ] **Processor 1**: `GenerateFlowFile` (trigger thủ công hoặc cron)
- [ ] **Processor 2**: `ExecuteSQL` — query tất cả published posts
- [ ] **Processor 3**: `ConvertRecord` — chuyển sang JSON
- [ ] **Processor 4**: `PutFile` — ghi ra file `posts_export_{date}.json`

---

## PHASE 8 — Frontend: Setup & Layout

### 8.1 Cấu hình Routing

- [ ] Tạo cấu trúc thư mục `src/`:
  ```
  pages/ · components/ · services/ · store/ · hooks/ · i18n/ · utils/ · types/
  ```
- [ ] Setup React Router DOM trong `App.tsx`:
  ```
  / → HomePage
  /login → LoginPage
  /register → RegisterPage
  /viet → PostEditorPage (new post) [protected]
  /bai-viet/:slug → PostDetailPage
  /u/:username → UserProfilePage
  /tag/:slug → TagPage
  /tim-kiem → SearchPage
  /dashboard → DashboardPage [protected]
  /dashboard/bai-viet → MyPostsPage [protected]
  /dashboard/luu → BookmarksPage [protected]
  /thong-bao → NotificationsPage [protected]
  ```
- [ ] Tạo **`ProtectedRoute`** component — redirect về `/login` nếu chưa đăng nhập

---

### 8.2 i18n Setup

- [ ] Tạo `src/i18n/index.ts` — config i18next:
  - Detect ngôn ngữ từ `localStorage` hoặc browser
  - Fallback: `vi` (tiếng Việt)
  - Resources load từ JSON files
- [ ] Tạo file translations:
  - `src/i18n/locales/vi/common.json` — Navigation, buttons, labels
  - `src/i18n/locales/vi/post.json` — Post-related text
  - `src/i18n/locales/vi/auth.json` — Login/Register labels
  - `src/i18n/locales/en/common.json`
  - `src/i18n/locales/en/post.json`
  - `src/i18n/locales/en/auth.json`
- [ ] Tạo **`LanguageToggle`** component — button chuyển VI/EN, lưu vào localStorage

---

### 8.3 Axios & React Query Setup

- [ ] Tạo `src/services/api.ts` — Axios instance:
  - `baseURL: import.meta.env.VITE_API_URL` (từ `.env`)
  - Request interceptor: tự động thêm `Authorization: Bearer {token}` từ store
  - Response interceptor: nếu 401 → thử refresh token → nếu fail → logout
- [ ] Setup `QueryClient` trong `main.tsx` với `QueryClientProvider`
- [ ] Cấu hình `staleTime: 5 * 60 * 1000` (5 phút) cho queries thường dùng

---

### 8.4 Zustand Auth Store

- [ ] Tạo `src/store/authStore.ts`:
  ```typescript
  interface AuthStore {
    user: User | null;
    accessToken: string | null;
    isAuthenticated: boolean;
    login(token, user): void;
    logout(): void;
    setUser(user): void;
  }
  ```
- [ ] Persist store vào localStorage (`zustand/middleware/persist`)

---

### 8.5 Layout Components

- [ ] **`Navbar`**: Logo, navigation links, search input, language toggle, user menu (avatar + dropdown) hoặc Login/Register buttons
- [ ] **`Sidebar`** (desktop only): Trending tags, Suggested users to follow
- [ ] **`Footer`**: Links, copyright
- [ ] **`MainLayout`**: Navbar + `<Outlet />` + Sidebar (nếu có)
- [ ] **`AuthLayout`**: Simple centered layout cho login/register

---

## PHASE 9 — Frontend: Auth Pages

### 9.1 Login & Register

- [ ] **`LoginPage`**:
  - Form với React Hook Form + Zod validation
  - Fields: email, password
  - Submit → gọi `authService.login()` → lưu token vào Zustand store → redirect `/`
  - Link "Chưa có tài khoản? Đăng ký"

- [ ] **`RegisterPage`**:
  - Fields: email, username, fullName, password, confirmPassword
  - Client-side validation: username không chứa khoảng trắng, password ≥ 8 ký tự
  - Submit → gọi `authService.register()` → auto login → redirect `/`

- [ ] **`authService.ts`**:
  - `login(email, password)` → `POST /api/auth/login` → trả về tokens + user
  - `register(data)` → `POST /api/auth/register`
  - `logout()` → `POST /api/auth/logout` + clear store
  - `refreshToken()` → `POST /api/auth/refresh`

---

## PHASE 10 — Frontend: Core Pages

### 10.1 Home Feed

- [ ] **`HomePage`**:
  - Tab switcher: "Xu hướng" / "Mới nhất" / "Đang theo dõi" (tab cuối chỉ hiện khi đã login)
  - Infinite scroll hoặc Load more pagination
  - Dùng `useInfiniteQuery` của React Query

- [ ] **`PostCard`** component:
  - Cover image, title, excerpt (truncate 2 dòng)
  - Author avatar + name, ngày đăng, thời gian đọc
  - Tags (hiển thị tối đa 3)
  - Upvote count, comment count, bookmark icon

---

### 10.2 Post Detail

- [ ] **`PostDetailPage`**:
  - SEO: cập nhật `<title>` và `<meta description>` động (react-helmet-async)
  - Hiển thị cover image, title, author info, ngày đăng
  - Render HTML content từ TipTap (dùng `dangerouslySetInnerHTML` hoặc `@tiptap/react` viewer mode)
  - Table of Contents sidebar (desktop)
  - Vote widget (upvote/downvote với animation)
  - Share button (copy link)
  - Bookmark button
  - Tags
  - Phần comments (CommentList + CommentForm)

- [ ] **`PostVote`** component:
  - Hiển thị số upvote/downvote
  - Click upvote → gọi API → optimistic update (cập nhật UI trước, rollback nếu lỗi)

- [ ] **`CommentSection`**:
  - Load top-level comments → hiển thị với avatar, content, thời gian
  - "Xem X replies" → load replies
  - Form reply inline

---

### 10.3 Post Editor

- [ ] **`PostEditorPage`**:
  - Có 2 mode: Tạo mới và Chỉnh sửa (dùng chung component)
  - Field title (input lớn, không border)
  - **TipTap Editor**: toolbar với Bold, Italic, H2, H3, Blockquote, Code, Bullet List, Image upload, Link
  - Sidebar settings: Tags selector (multi-select), Cover image upload, Excerpt input
  - Buttons: "Lưu nháp" và "Xuất bản"
  - Auto-save draft mỗi 30 giây (debounce)

- [ ] **TipTap Image Upload**: khi paste/upload ảnh → gọi API `POST /api/upload` → nhận URL → chèn vào editor
  > Backend cần endpoint upload ảnh: lưu vào local filesystem hoặc Cloudinary/S3

---

### 10.4 User Profile

- [ ] **`UserProfilePage`**:
  - Header: avatar, tên, bio, social links, số followers/following
  - Follow/Unfollow button (nếu không phải bản thân)
  - Tabs: Bài viết | Đã lưu (chỉ xem được của bản thân)
  - Danh sách bài viết dạng PostCard

---

### 10.5 Tag Page & Search

- [ ] **`TagPage`**: Hiển thị tên tag + số bài, danh sách PostCard
- [ ] **`SearchPage`**:
  - Search input với debounce 300ms
  - Tabs: Bài viết | Tác giả
  - Kết quả bài viết: PostCard
  - Kết quả tác giả: UserCard (avatar + name + follow button)

---

## PHASE 11 — Frontend: Dashboard

### 11.1 Dashboard Layout

- [ ] Sidebar navigation: Tổng quan | Bài viết của tôi | Đã lưu | Cài đặt tài khoản
- [ ] **`OverviewPage`**:
  - Số liệu tổng: Tổng views, Tổng upvotes, Tổng followers
  - Biểu đồ line chart (Recharts): views/upvotes theo 7 ngày / 30 ngày / 90 ngày
  - Danh sách top 5 bài viết có nhiều view nhất

- [ ] **`MyPostsPage`**:
  - Table bài viết: tiêu đề, trạng thái, views, upvotes, ngày đăng
  - Filter theo status (DRAFT | PUBLISHED | ARCHIVED)
  - Actions: Chỉnh sửa, Xuất bản, Xóa

- [ ] **`BookmarksPage`**: Danh sách bài đã bookmark (PostCard grid)

---

### 11.2 Notifications Page

- [ ] **`NotificationsPage`**:
  - Danh sách thông báo: icon theo loại (🔔 follow, 👍 upvote, 💬 comment)
  - Thông báo chưa đọc → highlight
  - Click → navigate đến bài viết/profile liên quan → đánh dấu đã đọc
- [ ] **Notification badge** trong Navbar: polling `GET /api/notifications/unread-count` mỗi 30 giây

---

## PHASE 12 — Hoàn thiện & Tài liệu

### 12.1 API Documentation (Swagger)

- [ ] Thêm annotations `@Operation`, `@ApiResponse`, `@Tag` vào tất cả controllers
- [ ] Thêm `@Schema` vào DTOs
- [ ] Truy cập `http://localhost:8080/swagger-ui.html` → kiểm tra đầy đủ
- [ ] Export OpenAPI spec ra `openapi.yaml` → commit vào repo

---

### 12.2 Error Handling

- [ ] **Backend**: `GlobalExceptionHandler` với `@RestControllerAdvice`:
  - `ResourceNotFoundException` → 404
  - `UnauthorizedException` → 401
  - `ForbiddenException` → 403
  - `MethodArgumentNotValidException` → 400 + list field errors
  - `Exception` (catch-all) → 500
- [ ] **Frontend**: Axios response interceptor hiển thị toast thông báo lỗi

---

### 12.3 README & Hướng dẫn Chạy

- [ ] Cập nhật `README.md` với:
  - Mô tả dự án, tech stack
  - **Hướng dẫn chạy local**:
    1. `cp .env.example .env` → điền giá trị
    2. `docker-compose up -d` → chờ services healthy
    3. `cd bookink-backend && mvn spring-boot:run`
    4. `cd bookink-frontend && npm run dev`
  - Link Swagger UI, Kafdrop UI, NiFi UI
  - Sơ đồ kiến trúc hệ thống

---

### 12.4 Kiểm tra E2E Cuối

- [ ] Đăng ký tài khoản mới → nhận JWT ✓
- [ ] Đăng nhập → token lưu trong store ✓
- [ ] Tạo bài viết với editor → publish ✓
- [ ] Bài xuất hiện trong feed Trending ✓
- [ ] Upvote bài → Kafka event gửi → Kafdrop hiển thị message ✓
- [ ] NiFi xử lý event → `post_analytics` có data ✓
- [ ] Comment bài → Notification tạo trong DB ✓
- [ ] Chuyển ngôn ngữ EN ↔ VI → giao diện đổi ngay ✓
- [ ] Dashboard tác giả: biểu đồ views hiển thị ✓
- [ ] Redis cache: post detail load lần 2 nhanh hơn lần 1 ✓

---

## Tóm tắt Tiến độ

| Phase | Mô tả | Trạng thái |
|---|---|---|
| Phase 0 | Setup dự án + Docker + Khởi tạo project | `[ ]` |
| Phase 1 | Backend: DB Schema + JPA Entities + Repository | `[ ]` |
| Phase 2 | Backend: Auth + JWT + Spring Security | `[ ]` |
| Phase 3 | Backend: Post CRUD + Tags + Search | `[ ]` |
| Phase 4 | Backend: Comments + Vote + Bookmark + Follow | `[ ]` |
| Phase 5 | Backend: Redis Cache + View Buffer + Rate Limit | `[ ]` |
| Phase 6 | Backend: Kafka Events + Consumers + Notifications | `[ ]` |
| Phase 7 | NiFi: Analytics ETL Pipeline | `[ ]` |
| Phase 8 | Frontend: Setup + i18n + Layout + Routing | `[ ]` |
| Phase 9 | Frontend: Auth Pages (Login + Register) | `[ ]` |
| Phase 10 | Frontend: Home + Post Detail + Editor + Profile | `[ ]` |
| Phase 11 | Frontend: Dashboard + Analytics + Notifications | `[ ]` |
| Phase 12 | Tài liệu + Swagger + Error Handling + E2E Test | `[ ]` |
