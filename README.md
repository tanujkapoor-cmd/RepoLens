# RepoLens 🔍
### AI Codebase Intelligence Platform

RepoLens is an intelligent, full-stack developer assistant that enables conversational exploration of GitHub repositories. By combining **Spring Boot 4.1**, **Next.js 16**, **PostgreSQL with pgvector**, and **Google Gemini** (via Spring AI's OpenAI-compatible client), RepoLens parses, indexes, and embeds codebases to provide accurate, context-aware answers with real-time streaming.

---

## 🌟 Key Features

- **GitHub OAuth2 Integration**: Secure sign-in with your GitHub account (`read:user, repo` scope). Personal access tokens are encrypted at rest using AES-256 before storage.
- **Automated Repository Indexing**: Fetches repository file trees via GitHub API, filters out binaries and vendor files, chunks code intelligently using `TokenTextSplitter`, and embeds chunks asynchronously.
- **RAG (Retrieval-Augmented Generation)**: Uses cosine similarity searches over PostgreSQL vector store with an HNSW index to fetch the most relevant code snippets for your queries.
- **Real-Time Token Streaming**: Streams AI responses token-by-token directly to the chat interface using Server-Sent Events (SSE).
- **Google Gemini Powered**: Designed to run on **Google Gemini** (`gemini-2.0-flash` and `text-embedding-004`), offering high performance and cost-effective AI capabilities without requiring paid OpenAI credits.
- **Modern UI**: Built with Next.js 16, React 19, Tailwind CSS v4, Lucide Icons, and TanStack React Query for smooth state management and caching.

---

## 🏗️ Architecture & Pipeline

```
                     +---------------------------------------+
                     |         Next.js 16 Frontend           |
                     |  (React 19, Tailwind v4, shadcn/ui)  |
                     +-------------------+-------------------+
                                         |
                       HTTP / SSE Stream | Cookie: REPOLENS_SESSION
                                         v
                     +---------------------------------------+
                     |         Spring Boot 4.1 API           |
                     |    (Spring Security OAuth2, MVC)      |
                     +---------+-------------------+---------+
                               |                   |
               Async Indexing  |                   | RAG & Chat
                               v                   v
+--------------------------------+       +------------------------------------+
|        GitHub API v3           |       |           Google Gemini            |
| (Tree, Blobs, Repositories)    |       | (gemini-2.0-flash, text-embed-004) |
+--------------------------------+       +------------------------------------+
                               \                   /
                                v                 v
                     +---------------------------------------+
                     |       PostgreSQL 16 + pgvector        |
                     |       (Port 5433, HNSW Index)         |
                     +---------------------------------------+
```

### 1. Indexing Pipeline
1. Fetch repository file tree via GitHub Tree API.
2. Filter out binaries, lock files, images, archives, and files exceeding size limit (`100 KB`).
3. Split file contents into chunks (`800 tokens` with `100 token overlap`).
4. Generate 768-dimensional vector embeddings in batches using Google's `text-embedding-004`.
5. Store embeddings and chunk metadata into PostgreSQL `vector_store` table.

### 2. RAG & Chat Pipeline
1. User asks a question within a repository chat session.
2. The question is embedded into a 768-dimensional vector.
3. Top-8 most relevant code chunks are retrieved via Cosine Distance matching on the HNSW vector index.
4. An augmented prompt containing retrieved code snippets and system instructions is constructed.
5. `gemini-2.0-flash` streams the answer via `SseEmitter` back to the client.

---

## 🛠️ Tech Stack

| Layer | Technology |
| :--- | :--- |
| **Backend** | Java 21, Spring Boot 4.1.0, Spring AI 2.0.0, Spring Security OAuth2, Spring Data JPA |
| **Frontend** | Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4, TanStack Query v5 |
| **AI / LLM** | Google Gemini (`gemini-2.0-flash` for chat, `text-embedding-004` for 768-dim embeddings) |
| **Database** | PostgreSQL 16 with `pgvector` extension (HNSW index, Cosine distance) |
| **Authentication** | GitHub OAuth2 (cookie-based session `REPOLENS_SESSION`, encrypted tokens at rest) |
| **Containerization** | Docker, Docker Compose |

---

## 📋 Prerequisites

Ensure you have the following installed on your machine:

- **Java Development Kit (JDK) 21** or later
- **Node.js 18+** and **npm** (or `pnpm` / `yarn`)
- **Docker & Docker Compose** (for PostgreSQL + pgvector)
- **Git**
- **Google Gemini API Key** (Get one for free at [Google AI Studio](https://aistudio.google.com/))
- **GitHub OAuth App credentials** (Client ID & Client Secret)

---

## ⚙️ Configuration & Environment Variables

### 1. GitHub OAuth App Setup
1. Go to [GitHub Developer Settings > OAuth Apps](https://github.com/settings/developers).
2. Click **New OAuth App**.
3. Fill in:
   - **Application name**: `RepoLens Local`
   - **Homepage URL**: `http://localhost:3000`
   - **Authorization callback URL**: `http://localhost:8080/login/oauth2/code/github`
4. Click **Register application**.
5. Note down the **Client ID** and generate a new **Client Secret**.

### 2. Environment Variables Summary

#### Backend Configuration

You can set these via environment variables or define them in your local `backend/src/main/resources/application.properties`:

| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `DB_URL` | `jdbc:postgresql://localhost:5433/repolens` | PostgreSQL connection URL |
| `DB_USERNAME` | `postgres` | Database username |
| `DB_PASSWORD` | `postgres` | Database password |
| `SPRING_AI_OPENAI_API_KEY` | *(Required)* | Your Google Gemini API Key |
| `GITHUB_CLIENT_ID` | *(Required)* | GitHub OAuth App Client ID |
| `GITHUB_CLIENT_SECRET` | *(Required)* | GitHub OAuth App Client Secret |
| `FRONTEND_URL` | `http://localhost:3000` | Frontend application URL |
| `CORS_ALLOWED_ORIGINS` | `http://localhost:3000` | Allowed CORS origins for API |
| `TOKEN_ENCRYPTOR_PASSWORD`| `repolens-local-encrypt-key-change-me` | AES key for encrypting GitHub tokens |
| `TOKEN_ENCRYPTOR_SALT` | `deadbeefcafebabe` | Salt for AES token encryption |

> **Note on Gemini Integration**: RepoLens uses Spring AI's OpenAI-compatible client pointed to:
> `https://generativelanguage.googleapis.com/v1beta/openai`
> Supply your Gemini API key in `spring.ai.openai.api-key`.

---

## 🚀 Quick Start Guide

### Step 1: Clone the Repository
```bash
git clone https://github.com/tanujkapoor-cmd/RepoLens.git
cd RepoLens
```

### Step 2: Start PostgreSQL with pgvector
Start the PostgreSQL container with the vector extension preloaded:
```bash
docker compose up -d
```
Verify the container is healthy:
```bash
docker ps
```
*(PostgreSQL will be exposed on host port `5433`)*

### Step 3: Run the Backend
Navigate to the `backend` directory:
```bash
cd backend
```

Set required environment variables:
```bash
# On Linux / macOS:
export SPRING_AI_OPENAI_API_KEY="your-gemini-api-key"
export GITHUB_CLIENT_ID="your-github-client-id"
export GITHUB_CLIENT_SECRET="your-github-client-secret"
export FRONTEND_URL="http://localhost:3000"
export CORS_ALLOWED_ORIGINS="http://localhost:3000"

# On Windows (PowerShell):
$env:SPRING_AI_OPENAI_API_KEY="your-gemini-api-key"
$env:GITHUB_CLIENT_ID="your-github-client-id"
$env:GITHUB_CLIENT_SECRET="your-github-client-secret"
$env:FRONTEND_URL="http://localhost:3000"
$env:CORS_ALLOWED_ORIGINS="http://localhost:3000"
```

Start the Spring Boot server:
```bash
./mvnw spring-boot:run
# Or on Windows:
.\mvnw.cmd spring-boot:run
```
The backend will start at `http://localhost:8080`.

### Step 4: Run the Frontend
Open a new terminal and navigate to the `client` directory:
```bash
cd client
npm install
npm run dev
```
The client will be running at `http://localhost:3000`.

---

## 📡 API Reference

### Authentication
- `GET /api/auth/login-url` — Get the GitHub OAuth login URL
- `GET /api/auth/me` — Get current authenticated user profile
- `POST /api/auth/logout` — Invalidate session and clear auth cookies

### Repositories & Indexing
- `GET /api/repos` — Fetch user's GitHub repositories
- `GET /api/repos/{id}` — Get details of a tracked repository
- `POST /api/repos/{id}/index` — Trigger async indexing of the repository
- `GET /api/repos/{id}/status` — Poll repository indexing status (`PENDING`, `INDEXING`, `COMPLETED`, `FAILED`)

### Chat & Streaming
- `POST /api/chat/sessions` — Create a new chat session for a repository
- `GET /api/chat/sessions` — List all chat sessions
- `GET /api/chat/sessions/{id}` — Get chat session details and message history
- `POST /api/chat/sessions/{id}/messages` — Send message and receive streamed response (`text/event-stream`)

---

## 📂 Project Structure

```
RepoLens/
├── backend/                        # Spring Boot 4.1 Backend
│   ├── src/main/java/devPilot/backend/
│   │   ├── config/                 # Security, AI, ThreadPool & Web configs
│   │   ├── controller/             # REST controllers (Auth, Repos, Chat)
│   │   ├── dto/                    # Request and response transfer objects
│   │   ├── entity/                 # JPA Entities (User, Repo, Session, Message)
│   │   ├── repository/             # Spring Data JPA repositories
│   │   ├── security/               # OAuth2 handlers, token encryption
│   │   └── services/
│   │       ├── ai/                 # RAG retrieval & streaming chat service
│   │       ├── github/             # GitHub API client (tree, blob fetcher)
│   │       └── indexing/           # Code parsing, chunking & vector store logic
│   ├── src/main/resources/
│   │   └── application.properties  # Backend configuration properties
│   └── pom.xml                     # Maven dependencies
│
├── client/                         # Next.js 16 Frontend
│   ├── src/
│   │   ├── app/                    # App Router pages (/login, /dashboard, /chat)
│   │   ├── components/             # UI components, layout, chat components
│   │   ├── hooks/                  # Custom React hooks (React Query & SSE streaming)
│   │   └── lib/                    # API client, types, utility functions
│   ├── package.json
│   └── next.config.ts
│
├── docker/
│   └── postgres/
│       └── init-extensions.sql     # Pre-initializes pgvector and uuid-ossp
├── docker-compose.yml              # PostgreSQL 16 + pgvector container definition
└── README.md
```

---

## 🛡️ Security Considerations

- **Secrets in `.gitignore`**: Sensitive credentials (`application.properties`, `.env.local`, API keys) must never be committed. Template files or environment variables should be used instead.
- **Token Encryption**: Access tokens received from GitHub are encrypted using AES with password and salt before being stored in the database.
- **CORS & Cookies**: Protected using `SameSite=Lax`, `HttpOnly` session cookies scoped to the configured frontend origin.

---

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

This project is open-source and available under the [MIT License](LICENSE).
