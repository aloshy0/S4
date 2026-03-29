# My Tiptap Project

A full-stack, secure note-taking and e-learning platform with authenticated video streaming. Built with **Next.js 16**, **Tiptap v3**, **Go**, **Cloudflare Workers**, and **Cloudflare R2**.

The core idea: users log in, create rich-text notes with an advanced WYSIWYG editor, and embed videos stored in Cloudflare R2. **Videos are only accessible to authenticated users** — raw URLs are useless without a valid, time-limited HMAC token.

---

## 🏗 High-Level Architecture

```mermaid
graph TB
    subgraph Client["🖥 Browser"]
        A[Tiptap Editor]
    end

    subgraph NextJS["⚡ Next.js (port 3000)"]
        B[Pages & UI]
        C[NextAuth Session]
        D["/api/video/sign"]
        E[Server Actions]
        F[Prisma ORM]
    end

    subgraph Go["🔧 Go Backend (port 8080)"]
        G[JWT Verification]
        H[HMAC URL Signing]
    end

    subgraph CF["☁️ Cloudflare"]
        I[Worker - Token Validator]
        J[R2 Bucket - Video Storage]
    end

    subgraph DB["🗄 PostgreSQL"]
        K[(Users & Notes)]
    end

    A --> B
    B --> C
    A -- "auto-save" --> E
    E --> F --> K
    A -- "embed video" --> D
    D -- "mint JWT" --> G
    G -- "sign URL" --> H
    H -- "signed URL" --> D --> A
    A -- "play video" --> I
    I -- "validated" --> J
    I -- "stream" --> A
```

The system is split into **3 independent services**, each with a single responsibility:

| Service | Port | Tech | Responsibility |
|---------|------|------|---------------|
| **Frontend** | `3000` | Next.js 16, React 19, Tiptap v3 | UI, auth, note CRUD, API proxy |
| **Signing Backend** | `8080` | Go (stdlib + golang-jwt) | JWT verification, HMAC URL signing |
| **Edge Gateway** | `8787` | Cloudflare Worker | Token validation, R2 video serving |

---

## 📂 Project Structure

```text
my-tiptap-project/
├── app/                          # Next.js App Router
│   ├── api/
│   │   ├── auth/[...nextauth]/   #   NextAuth route handler
│   │   └── video/sign/           #   Video signing API proxy (session-gated)
│   ├── dashboard/                #   Notes dashboard (SSR, auth-protected)
│   ├── login/                    #   Login page
│   ├── signup/                   #   Signup page
│   ├── notes/[id]/               #   Note editor page + editor wrapper
│   ├── layout.tsx                #   Root layout
│   └── page.tsx                  #   Root redirect (→ /dashboard or /login)
│
├── actions/                      # React Server Actions
│   ├── auth.ts                   #   loginAction(), registerAction()
│   └── notes.ts                  #   createNote(), updateNote()
│
├── backend/                      # Go Microservice
│   ├── main.go                   #   HTTP server entry point
│   ├── auth/jwt.go               #   Real JWT verification (golang-jwt/jwt/v5)
│   ├── handlers/video.go         #   HMAC signing + CORS handler
│   └── go.mod                    #   Go module (golang-jwt dependency)
│
├── components/
│   ├── tiptap-node/              #   Custom Tiptap node extensions
│   │   ├── r2-video-node/        #     R2Video node (renders <video> in editor)
│   │   ├── image-upload-node/    #     Drag-and-drop image uploads
│   │   ├── heading-node/         #     Styled headings
│   │   └── ...                   #     blockquote, code-block, list, etc.
│   ├── tiptap-ui/                #   Toolbar button components
│   │   ├── r2-video-button/      #     Video embed button + useR2Video hook
│   │   └── ...                   #     heading, list, mark, link, etc.
│   ├── tiptap-templates/
│   │   └── simple/               #   SimpleEditor (main editor component)
│   ├── tiptap-ui-primitive/      #   Base UI primitives (Button, Toolbar)
│   └── tiptap-icons/             #   SVG icon components
│
├── hooks/                        # Custom React hooks
│   ├── use-tiptap-editor.ts      #   Editor instance hook
│   ├── use-cursor-visibility.ts  #   Cursor visibility management
│   └── ...                       #   breakpoint, scrolling, window-size, etc.
│
├── lib/                          # Shared utilities
│   ├── auth.ts                   #   NextAuth config (credentials, JWT callbacks)
│   ├── prisma.ts                 #   Prisma client (PG adapter, singleton)
│   └── tiptap-utils.ts           #   Image upload handler, size limits
│
├── prisma/
│   └── schema.prisma             #   Database schema (User, Note models)
│
├── worker/                       # Cloudflare Worker
│   ├── src/index.js              #   Token validation + R2 serving + Range support
│   └── wrangler.toml             #   Worker config (R2 binding, HMAC secret)
│
├── .env                          #   Environment variables (secrets, DB URL)
├── seed.ts                       #   Database seeder (creates test user)
└── package.json                  #   Frontend dependencies
```

---

## 🔒 Authentication System

### User Authentication (NextAuth.js)

```mermaid
sequenceDiagram
    participant Browser
    participant NextJS as Next.js Server
    participant DB as PostgreSQL

    Browser->>NextJS: POST /login (email, password)
    NextJS->>DB: Find user by email
    DB-->>NextJS: User record (hashed password)
    NextJS->>NextJS: bcrypt.compare(input, hash)
    alt Password matches
        NextJS->>NextJS: Create JWT session token
        NextJS->>NextJS: Set HTTP-only cookie
        NextJS-->>Browser: 303 Redirect → /dashboard
    else Password fails
        NextJS-->>Browser: "Invalid email or password"
    end
```

**How it works:**

1. User submits email + password via `loginAction()` server action
2. NextAuth's `authorize()` callback queries PostgreSQL via Prisma
3. Password is verified using `bcrypt.compare()`
4. On success, NextAuth creates a **JWT session token** stored in an **HTTP-only cookie**
5. The JWT contains the user's `id` (added via custom `jwt` and `session` callbacks in `lib/auth.ts`)
6. Every protected page calls `auth()` server-side to validate the session

**Session flow through the app:**
```
/ (root) → auth() → session exists? → /dashboard : /login
/dashboard → auth() → no session? → redirect /login
/notes/[id] → auth() → no session? → redirect /login
/api/video/sign → auth() → no session? → 401 Unauthorized
```

### Password Storage

```
User types: "mypassword"
        ↓
bcrypt.hash("mypassword", 10)  →  "$2a$10$xK7Jz..."  (stored in DB)
        ↓
Login: bcrypt.compare("mypassword", "$2a$10$xK7Jz...")  →  true ✅
```

Passwords are **never stored in plaintext**. bcrypt uses a cost factor of 10 (adaptive hashing).

---

## 🎥 Secure Video Streaming Architecture

This is the heart of the project. The goal: **only authenticated users can watch videos, even if someone shares the raw video URL.**

### The Complete Video Flow

```mermaid
sequenceDiagram
    participant User as 👤 User (Browser)
    participant Next as ⚡ Next.js API
    participant Go as 🔧 Go Backend
    participant Worker as ☁️ CF Worker
    participant R2 as 📦 R2 Bucket

    Note over User: User clicks Video icon in editor
    User->>User: Prompt: "Enter video path"
    User->>Next: GET /api/video/sign?path=videos/lesson.mp4
    Note over Next: Cookie sent automatically

    rect rgb(40, 40, 80)
        Note over Next: Step 1: Session Gate
        Next->>Next: auth() — validate NextAuth session
        alt No session
            Next-->>User: 401 Unauthorized
        end
    end

    rect rgb(40, 80, 40)
        Note over Next: Step 2: Mint Inter-Service JWT
        Next->>Next: Sign JWT {sub: userId, exp: 60s}
        Next->>Next: Using BACKEND_JWT_SECRET (HS256)
    end

    rect rgb(80, 40, 40)
        Note over Go: Step 3: Verify & Sign
        Next->>Go: GET /api/video/sign?path=... + Bearer JWT
        Go->>Go: Verify JWT (golang-jwt/jwt/v5)
        alt Invalid/expired JWT
            Go-->>Next: 401 Unauthorized
        end
        Go->>Go: HMAC-SHA256 sign: "videos/lesson.mp4:expiry"
        Go->>Go: Using VIDEO_HMAC_SECRET
        Go->>Go: Base64URL encode token
        Go-->>Next: { url: "localhost:8787/videos/lesson.mp4?token=abc123" }
    end

    Next-->>User: { url: "...?token=abc123" }
    User->>User: Insert <video src="...?token=abc123">

    rect rgb(80, 60, 20)
        Note over Worker: Step 4: Token Validation at Edge
        User->>Worker: GET /videos/lesson.mp4?token=abc123
        Worker->>Worker: Decode Base64URL token
        Worker->>Worker: Extract: path + expiry + signature
        Worker->>Worker: Verify path matches request
        Worker->>Worker: Verify not expired
        Worker->>Worker: Recompute HMAC-SHA256 with SECRET_KEY
        Worker->>Worker: Compare signatures (constant-time)
        alt Token invalid
            Worker-->>User: 401 Unauthorized
        end
    end

    rect rgb(20, 60, 80)
        Note over R2: Step 5: Serve from R2
        Worker->>R2: env.MY_BUCKET.get("videos/lesson.mp4")
        alt File not found
            Worker-->>User: 404 Not Found
        end
        R2-->>Worker: Video binary stream
        Worker-->>User: 200 OK (video/mp4) + Range support
    end

    Note over User: Video plays in editor ▶️
```

### Why 3 Services?

| Layer | Why it exists |
|-------|--------------|
| **Next.js API** `/api/video/sign` | **Session boundary** — only place that has access to NextAuth cookies. Keeps the browser from ever knowing any secrets. |
| **Go Backend** `/api/video/sign` | **Crypto performance** — HMAC signing in Go's compiled binary is ~10x faster than Node.js. Decoupled so it can scale independently. |
| **Cloudflare Worker** | **Edge validation** — runs in 300+ data centers worldwide. Validates tokens in <1ms. Serves video from R2 without hitting your origin server. |

### What Each Secret Does

```mermaid
graph LR
    subgraph Secrets
        A["AUTH_SECRET"]
        B["BACKEND_JWT_SECRET"]
        C["VIDEO_HMAC_SECRET"]
    end

    subgraph Services
        D["Next.js"]
        E["Go Backend"]
        F["CF Worker"]
    end

    A --> D
    B --> D
    B --> E
    C --> E
    C --> F

    style A fill:#4a9eff,color:#fff
    style B fill:#ff9f43,color:#fff
    style C fill:#ee5a24,color:#fff
```

| Secret | Shared Between | Purpose |
|--------|---------------|---------|
| `AUTH_SECRET` | Next.js only | Encrypts NextAuth JWT session cookies |
| `BACKEND_JWT_SECRET` | Next.js ↔ Go | Signs inter-service JWTs (60s lifetime) |
| `VIDEO_HMAC_SECRET` | Go ↔ Worker | Signs video URLs with HMAC-SHA256 (15min lifetime) |

### HMAC Token Structure

```
Token = Base64URL( "video/path:expiry_timestamp:hmac_signature" )

Example decoded:
  Path:      videos/lesson-1.mp4
  Expiry:    1711411200 (Unix timestamp, 15 min from now)
  Signature: a3f8c2...9d1e (HMAC-SHA256 of "videos/lesson-1.mp4:1711411200")
```

The Worker recomputes the HMAC independently. If the signature matches and the timestamp hasn't expired, the video is served. The token is **path-bound** — a token for `lesson-1.mp4` cannot be used to access `lesson-2.mp4`.

---

## ✏️ Editor & Auto-Save System

### Tiptap Editor Architecture

```mermaid
graph TB
    subgraph SimpleEditor
        A[useEditor Hook]
        B[Extensions]
        C[Toolbar]
        D[EditorContent]
    end

    subgraph Extensions
        E[StarterKit]
        F[Image / ImageUpload]
        G[R2Video]
        H[Highlight / Typography]
        I[TextAlign / Lists]
    end

    subgraph AutoSave
        J[onUpdate callback]
        K[1.5s debounce timer]
        L["updateNote() server action"]
        M[(PostgreSQL)]
    end

    A --> B & C & D
    B --> E & F & G & H & I
    D -- "user types" --> J
    J --> K
    K -- "after 1.5s idle" --> L
    L --> M
```

**How auto-save works:**

1. Every keystroke triggers Tiptap's `onUpdate` callback
2. The callback resets a **1.5-second debounce timer**
3. After 1.5s of no typing, the callback fires `updateNote(noteId, html)`
4. `updateNote` is a React Server Action that:
   - Validates the NextAuth session
   - Checks the user owns the note (`userId` match)
   - Updates the `content` column via Prisma
5. On next page load, `initialContent` is passed from the server to the editor

### Custom Tiptap Extensions

| Extension | Type | What it does |
|-----------|------|-------------|
| `R2Video` | Node | Renders a `<video>` element with `data-type="r2-video"`. Stores `src` attribute (the signed URL). |
| `ImageUploadNode` | Node | Drag-and-drop image upload with progress indicator. |
| `HorizontalRule` | Node | Custom styled `<hr>` element. |
| `StarterKit` | Bundle | Headings, paragraphs, bold, italic, strike, code, blockquote, lists. |
| `Highlight` | Mark | Multi-color text highlighting. |
| `TextAlign` | Extension | Left, center, right, justify alignment. |

---

## 🗄 Database Schema

```mermaid
erDiagram
    User ||--o{ Note : "has many"

    User {
        string id PK "cuid()"
        string email UK "unique"
        string password "bcrypt hash"
    }

    Note {
        string id PK "cuid()"
        string title "default: Untitled Note"
        string content "nullable, HTML from Tiptap"
        string userId FK "→ User.id"
        datetime createdAt "auto"
        datetime updatedAt "auto"
    }
```

**Cascade delete**: When a user is deleted, all their notes are deleted too (`onDelete: Cascade`).

---

## 📦 Key Dependencies

### Frontend (Next.js)

| Package | Version | Purpose |
|---------|---------|---------|
| `next` | 16.2.1 | React framework (SSR, API routes, App Router) |
| `react` | 19.2.4 | UI library |
| `@tiptap/react` | 3.20.5 | Headless WYSIWYG editor |
| `@tiptap/starter-kit` | 3.20.5 | Core editor extensions bundle |
| `next-auth` | 5.0.0-beta.30 | Authentication (credentials + JWT sessions) |
| `prisma` | 7.5.0 | Database ORM |
| `bcryptjs` | 3.0.3 | Password hashing |
| `jose` | (transitive) | JWT signing for inter-service auth |

### Backend (Go)

| Package | Purpose |
|---------|---------|
| `net/http` (stdlib) | HTTP server |
| `crypto/hmac` + `crypto/sha256` (stdlib) | HMAC-SHA256 URL signing |
| `github.com/golang-jwt/jwt/v5` | JWT parsing and verification |

### Cloudflare Worker

| Tool | Purpose |
|------|---------|
| `wrangler` | CLI for Worker dev/deploy |
| Web Crypto API (built-in) | HMAC-SHA256 token verification |

---

## 🛠 How to Run Locally

### Prerequisites
- Node.js 20+
- Go 1.21+
- PostgreSQL database (local or cloud like Supabase/Neon)
- Cloudflare account (for R2 bucket + Worker)

### 1. Database Setup

Create a `.env` file in the project root:

```env
# Database
DATABASE_URL="postgresql://user:password@localhost:5432/tiptap_db?schema=public"

# NextAuth session encryption
AUTH_SECRET="<generate with: node -e \"console.log(require('crypto').randomBytes(32).toString('base64'))\">"

# Inter-service JWT signing (shared: Next.js ↔ Go)
BACKEND_JWT_SECRET="<generate with same command>"

# Video URL HMAC signing (shared: Go ↔ Cloudflare Worker)
VIDEO_HMAC_SECRET="<generate with: node -e \"console.log(require('crypto').randomBytes(32).toString('hex'))\">"
```

Push the schema:
```bash
npx prisma db push
```

Seed a test user:
```bash
npx tsx seed.ts
# Creates: admin@example.com / password123
```

### 2. Start All 3 Services

**Terminal 1 — Next.js Frontend:**
```bash
npm install
npm run dev
```

**Terminal 2 — Go Backend** (needs env vars):
```bash
cd backend
BACKEND_JWT_SECRET="your-secret-here" \
VIDEO_HMAC_SECRET="your-hmac-secret-here" \
go run main.go
```

**Terminal 3 — Cloudflare Worker:**
```bash
cd worker
npx wrangler dev
```

> **Important:** The `VIDEO_HMAC_SECRET` in the Go command must match the `SECRET_KEY` in `worker/wrangler.toml`.

### 3. Configure R2 Bucket

1. Create an R2 bucket in the [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Upload video files (e.g., `videos/sample_video.mp4`)
3. Update `worker/wrangler.toml`:
   ```toml
   [[r2_buckets]]
   binding = "MY_BUCKET"
   bucket_name = "your-bucket-name"
   remote = true    # connects to real R2 in local dev
   ```
4. Run `wrangler login` to authenticate

---

## 🎯 Testing the Workflow

1. Navigate to `http://localhost:3000/login`
2. Log in with `admin@example.com` / `password123`
3. Click **+ New Note** on the dashboard
4. Type some content — it **auto-saves** after 1.5s of idle
5. Click the **📹 Video icon** in the toolbar
6. Enter a video path from your R2 bucket (e.g., `videos/sample_video.mp4`)
7. The video should embed and play in the editor

### Verify Security

Open an **incognito window** and try:

| Test | Expected Result |
|------|----------------|
| Visit `http://localhost:3000/api/video/sign?path=videos/test.mp4` | `401 Unauthorized` (no session) |
| Visit `http://localhost:8787/videos/test.mp4` | `401 Unauthorized` (no token) |
| Visit `http://localhost:8787/videos/test.mp4?token=fake` | `401 Unauthorized` (invalid token) |
| Wait 15+ minutes, try a previously valid signed URL | `401 Unauthorized` (expired) |

---

## 🚀 Deployment

### Cloudflare Worker
```bash
cd worker
npx wrangler deploy
```

### Go Backend
Build a static binary and deploy to any Linux server:
```bash
cd backend
CGO_ENABLED=0 go build -o video-signer .
# Run with env vars on your server
```

### Next.js
Deploy to Vercel, or build and self-host:
```bash
npm run build
npm start
```

> **Production checklist:**
> - [ ] Set real `AUTH_SECRET`, `BACKEND_JWT_SECRET`, `VIDEO_HMAC_SECRET`
> - [ ] Move `SECRET_KEY` in Worker from `[vars]` to `wrangler secret put SECRET_KEY`
> - [ ] Update CORS in `video.go` from `localhost:3000` to your production domain
> - [ ] Update `ALLOWED_DOMAIN` in `wrangler.toml` to your production domain
> - [ ] Update `CDN_URL` in `video.go` to your Worker's production URL
