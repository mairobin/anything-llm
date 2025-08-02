# AnythingLLM Server

A comprehensive Node.js server application for managing LLM (Large Language Model) workspaces, document processing, and AI chat functionality with multi-user support and extensive API capabilities.

## Server Overview

AnythingLLM Server is the backend component of a full-featured AI workspace management platform. It provides:

- **Workspace Management**: Create and manage AI workspaces with custom configurations
- **Document Processing**: Upload, embed, and manage documents for RAG (Retrieval-Augmented Generation)
- **Multi-LLM Support**: Integration with 25+ LLM providers (OpenAI, Anthropic, Ollama, etc.)
- **Vector Database Integration**: Support for multiple vector databases (LanceDB, Pinecone, Chroma, etc.)
- **Real-time Chat**: WebSocket-based streaming chat with workspaces
- **Multi-user System**: Role-based access control with admin, manager, and user roles
- **Embedding System**: Public embeddable chat widgets
- **Agent Flows**: Custom AI agent workflows and automations
- **API Integration**: Comprehensive REST API for external integrations

## Project Structure

```
server/
├── endpoints/          # API route definitions
│   ├── admin.js       # Admin management endpoints
│   ├── agentFlows.js  # Agent workflow endpoints
│   ├── chat.js        # Chat functionality endpoints
│   ├── system.js      # System configuration endpoints
│   ├── workspaces.js  # Workspace management endpoints
│   ├── api/           # Developer API endpoints (/v1/*)
│   ├── embed/         # Public embed endpoints
│   └── mobile/        # Mobile app specific endpoints
├── models/             # Database models and business logic
│   ├── user.js        # User management model
│   ├── workspace.js   # Workspace model
│   ├── documents.js   # Document management model
│   └── ...            # Additional models
├── utils/              # Utility modules and services
│   ├── AiProviders/   # LLM provider integrations
│   ├── EmbeddingEngines/ # Embedding model providers
│   ├── agents/        # AI agent implementations
│   ├── middleware/    # Authentication and validation middleware
│   ├── vectorDbProviders/ # Vector database integrations
│   └── chats/         # Chat processing utilities
├── prisma/            # Database schema and migrations
│   ├── schema.prisma  # Database schema definition
│   ├── migrations/    # Database migration files
│   └── seed.js        # Database seeding script
├── public/            # Static assets and frontend files
├── storage/           # File storage directory
│   ├── documents/     # Uploaded documents
│   ├── models/        # Downloaded AI models
│   ├── vector-cache/  # Vector embeddings cache
│   └── assets/        # System assets
├── jobs/              # Background job definitions
├── swagger/           # API documentation
├── index.js           # Server entry point
├── package.json       # Dependencies and scripts
└── .env.example       # Environment configuration template
```

## Technology Stack

### Core Framework
- **Node.js**: v18.12.1+ (JavaScript runtime)
- **Express.js**: Web application framework
- **WebSocket**: Real-time communication via `@mintplex-labs/express-ws`

### Database & ORM
- **Prisma**: Database ORM/query builder
- **SQLite**: Default database (production-ready)
- **PostgreSQL**: Optional database (via configuration)

### Authentication & Security
- **JWT**: JSON Web Tokens for session management
- **bcrypt**: Password hashing
- **CORS**: Cross-origin resource sharing
- **Encryption**: AES encryption for sensitive data

### AI & ML Integrations
- **LangChain**: AI framework for LLM integrations
- **Transformers**: Local embedding models via `@xenova/transformers`
- **25+ LLM Providers**: OpenAI, Anthropic, Ollama, Azure, Bedrock, etc.
- **Vector Databases**: LanceDB, Pinecone, Chroma, Qdrant, Weaviate, Milvus

### File Processing
- **Multer**: File upload handling
- **PDF Processing**: Document text extraction
- **OCR**: Image and PDF text recognition
- **Text Splitting**: Document chunking for embeddings

### Background Processing
- **Bree**: Job scheduling and background tasks
- **Graceful Shutdown**: Process management

## Server Configuration

### Environment Variables

The server uses environment variables for configuration. Key settings include:

```env
# Server Configuration
SERVER_PORT=3001
NODE_ENV=production

# Authentication
JWT_SECRET="your-jwt-secret-32-chars-min"
AUTH_TOKEN="your-auth-token"  # Single-user mode password

# LLM Provider Selection
LLM_PROVIDER='openai'
OPEN_AI_KEY=sk-your-openai-key
OPEN_MODEL_PREF='gpt-4o'

# Vector Database
VECTOR_DB="lancedb"  # or pinecone, chroma, qdrant, etc.

# Embedding Engine
EMBEDDING_ENGINE='native'
EMBEDDING_MODEL_PREF='Xenova/all-MiniLM-L6-v2'

# Multi-user Mode
# Enable for team/organization use
# Requires user accounts and role-based access

# HTTPS Support
ENABLE_HTTPS="true"
HTTPS_CERT_PATH="sslcert/cert.pem"
HTTPS_KEY_PATH="sslcert/key.pem"
```

### Database Configuration

The server uses Prisma with SQLite by default. To use PostgreSQL:

1. Update `prisma/schema.prisma`:
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

2. Set DATABASE_URL environment variable:
```env
DATABASE_URL="postgresql://user:password@localhost:5432/anythingllm"
```

3. Run migrations:
```bash
yarn prisma:setup
```

### Server Initialization

The server starts with the following sequence:

1. **Environment Loading**: Load `.env` file based on NODE_ENV
2. **Logging Setup**: Initialize Winston logger
3. **Express Configuration**: Setup middleware, CORS, body parsing
4. **SSL/HTTP Setup**: Configure HTTPS or HTTP based on settings
5. **Route Registration**: Mount all API endpoints
6. **Static Assets**: Serve frontend files (production only)
7. **WebSocket Setup**: Initialize real-time communication
8. **Graceful Shutdown**: Setup process signal handlers

## API Architecture

### Route Organization

The API is organized into two main layers:

#### Web UI API (`/api/*`)
Used by the frontend interface with JWT authentication:
- `/api/system/*` - System configuration and management
- `/api/admin/*` - Administrative functions
- `/api/workspace/*` - Workspace operations
- `/api/chat/*` - Chat functionality
- `/api/document/*` - Document management

#### Developer API (`/v1/*`)
RESTful API for external integrations with API key authentication:
- `/v1/auth` - Authentication validation
- `/v1/admin/*` - Administrative operations
- `/v1/workspace/*` - Workspace management
- `/v1/system/*` - System information

### Middleware Implementation

#### Authentication Middleware
- **`validatedRequest`**: JWT token validation for web sessions
- **`validApiKey`**: API key validation for developer API
- **`multiUserProtected`**: Enforces multi-user mode requirements

#### Authorization Middleware  
- **`flexUserRoleValid([roles])`**: Role-based access control
- **`validWorkspace`**: Workspace ownership validation
- **`adminRequired`**: Admin-only access enforcement

#### Feature Middleware
- **`embedMiddleware`**: Embed-specific request handling
- **`chatHistoryViewable`**: Chat history access control
- **`featureFlagEnabled`**: Feature toggle enforcement

### Error Handling

The server implements centralized error handling:

```javascript
// Global error handler
app.use((error, req, res, next) => {
  console.error(error.stack);
  res.status(500).json({
    error: "Internal server error",
    message: process.env.NODE_ENV === "development" ? error.message : undefined
  });
});

// 404 handler
app.all("*", (req, res) => {
  res.status(404).json({ error: "Endpoint not found" });
});
```

### Authentication System

#### Single-User Mode
- Uses AUTH_TOKEN environment variable
- JWT-based session management
- Suitable for personal deployments

#### Multi-User Mode
- Database-backed user accounts
- Role-based access control (Admin, Manager, Default)
- User suspension and daily message limits
- Password complexity validation

#### API Key Authentication
- Separate API keys for developer integrations
- Admin-generated and managed
- Bearer token format: `Authorization: Bearer sk-...`

## Database Design

### Core Models

#### Users (`users`)
```javascript
{
  id: number,
  username: string,
  password: string (hashed),
  role: string, // "admin", "manager", "default"
  suspended: boolean,
  dailyMessageLimit: number,
  bio: string,
  createdAt: datetime,
  lastUpdatedAt: datetime
}
```

#### Workspaces (`workspaces`)
```javascript
{
  id: number,
  name: string,
  slug: string, // URL-friendly identifier
  openAiPrompt: string, // System prompt
  openAiTemp: float, // Temperature setting
  openAiHistory: number, // Chat history length
  similarityThreshold: float, // Vector search threshold
  chatProvider: string, // LLM provider
  chatModel: string, // Model name
  topN: number, // Retrieved document count
  chatMode: string, // "chat" or "query"
  agentProvider: string,
  agentModel: string,
  vectorSearchMode: string // "default" or "rerank"
}
```

#### Documents (`workspace_documents`)
```javascript
{
  id: number,
  docId: string, // Unique document identifier
  filename: string,
  docpath: string, // File path
  workspaceId: number,
  metadata: string, // JSON metadata
  pinned: boolean,
  watched: boolean, // Auto-sync monitoring
  createdAt: datetime,
  lastUpdatedAt: datetime
}
```

#### Chat History (`workspace_chats`)
```javascript
{
  id: number,
  workspaceId: number,
  prompt: string, // User message
  response: string, // AI response
  include: boolean, // Include in chat history
  user_id: number,
  thread_id: number,
  api_session_id: string,
  feedbackScore: boolean,
  createdAt: datetime
}
```

### Relationships

- **Users ↔ Workspaces**: Many-to-many via `workspace_users`
- **Workspaces → Documents**: One-to-many
- **Workspaces → Chats**: One-to-many
- **Users → Chats**: One-to-many
- **Workspaces → Threads**: One-to-many
- **Users → API Keys**: One-to-many

### Database Migrations

Prisma manages database schema changes through migrations:

```bash
# Generate migration
npx prisma migrate dev --name migration_name

# Apply migrations
npx prisma migrate deploy

# Reset database
npx prisma migrate reset
```

## API Endpoints Documentation

### Authentication

All API endpoints require authentication via JWT tokens (web UI) or API keys (developer API).

#### Headers
```javascript
// Web UI Authentication
Authorization: Bearer <jwt_token>

// Developer API Authentication  
Authorization: Bearer <api_key>
```

### System Management

#### Health Check
```http
GET /api/ping
```
**Response**: `{ message: "pong" }`

#### System Status
```http
GET /api/setup-complete
```
**Response**: `{ multiUser: boolean, vectorDB: string }`

### User Management

#### Create User (Admin only)
```http
POST /api/admin/users/new
Content-Type: application/json

{
  "username": "john_doe",
  "password": "secure_password",
  "role": "default"
}
```

#### Get Users
```http
GET /api/admin/users
```

### Workspace Management

#### Create Workspace
```http
POST /api/workspace/new
Content-Type: application/json

{
  "name": "My Workspace",
  "openAiPrompt": "You are a helpful assistant...",
  "openAiTemp": 0.7,
  "chatProvider": "openai",
  "chatModel": "gpt-4o"
}
```

#### Get Workspace
```http
GET /api/workspace/:slug
```

#### Update Workspace
```http
POST /api/workspace/:slug/update
Content-Type: application/json

{
  "openAiTemp": 0.8,
  "similarityThreshold": 0.3
}
```

### Document Management

#### Upload Document
```http
POST /api/workspace/:slug/upload
Content-Type: multipart/form-data

file: <file_data>
```

#### Update Document Embeddings
```http
POST /api/workspace/:slug/update-embeddings
Content-Type: application/json

{
  "adds": ["doc-id-1", "doc-id-2"],
  "deletes": ["doc-id-3"]
}
```

### Chat Operations

#### Stream Chat
```http
POST /api/workspace/:slug/stream-chat
Content-Type: application/json

{
  "message": "Hello, how can you help me?",
  "mode": "chat"
}
```

**Response**: Server-sent events stream
```
data: {"type": "textResponseChunk", "textResponse": "Hello!"}
data: {"type": "textResponseChunk", "textResponse": " I can help"}
data: {"close": true, "error": false}
```

#### Get Chat History
```http
GET /api/workspace/:slug/chats
```

### Developer API Examples

#### List Workspaces
```http
GET /v1/workspaces
Authorization: Bearer sk-your-api-key
```

#### Chat with Workspace
```http
POST /v1/workspace/:slug/chat
Authorization: Bearer sk-your-api-key
Content-Type: application/json

{
  "message": "What is machine learning?",
  "mode": "chat"
}
```

## Development Setup

### Prerequisites

- **Node.js**: v18.12.1 or higher
- **npm** or **yarn**: Package manager
- **Git**: Version control

### Installation

1. **Clone the repository**:
```bash
git clone https://github.com/Mintplex-Labs/anything-llm.git
cd anything-llm/server
```

2. **Install dependencies**:
```bash
npm install
# or
yarn install
```

3. **Environment setup**:
```bash
cp .env.example .env
# Edit .env with your configuration
```

4. **Database setup**:
```bash
# Generate Prisma client
npx prisma generate

# Run database migrations
npx prisma migrate dev

# Seed database (optional)
npx prisma db seed
```

5. **Start development server**:
```bash
npm run dev
# or
yarn dev
```

The server will start on `http://localhost:3001` with hot reloading enabled.

### Development Commands

```bash
# Development with auto-restart
npm run dev

# Production build
npm run start

# Database operations
npx prisma studio          # Database GUI
npx prisma migrate dev      # Create migration
npx prisma migrate reset    # Reset database

# Code formatting
npm run lint               # Format code with Prettier

# API documentation
npm run swagger           # Generate Swagger docs
```

### Environment Setup

#### Required Environment Variables

```env
# Core Settings
SERVER_PORT=3001
JWT_SECRET="your-secure-jwt-secret-minimum-32-characters"

# Choose authentication mode
AUTH_TOKEN="single-user-password"  # Single-user mode
# OR enable multi-user mode (leave AUTH_TOKEN unset)

# LLM Configuration (choose one)
LLM_PROVIDER='openai'
OPEN_AI_KEY=sk-your-openai-api-key
OPEN_MODEL_PREF='gpt-4o'

# Vector Database (default: LanceDB)
VECTOR_DB="lancedb"

# Embedding Engine (default: local)
EMBEDDING_ENGINE='native'
EMBEDDING_MODEL_PREF='Xenova/all-MiniLM-L6-v2'
```

#### Optional Configuration

```env
# Whisper (Speech-to-Text)
WHISPER_PROVIDER="local"

# Text-to-Speech
TTS_PROVIDER="native"

# Agent Search Capabilities
AGENT_SERPER_DEV_KEY=your-serper-key
AGENT_SEARXNG_API_URL=http://localhost:8888

# Password Complexity (multi-user mode)
PASSWORDMINCHAR=8
PASSWORDREQUIREMENTS=2
```

### Database Setup Instructions

#### Using SQLite (Default)
No additional setup required. Database file is created automatically at `storage/anythingllm.db`.

#### Using PostgreSQL
1. Install and start PostgreSQL
2. Create database:
```sql
CREATE DATABASE anythingllm;
```
3. Update `.env`:
```env
DATABASE_URL="postgresql://username:password@localhost:5432/anythingllm"
```
4. Update `prisma/schema.prisma`:
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```
5. Run migrations:
```bash
npx prisma migrate dev
```

## Code Architecture

### Design Patterns

#### Model-View-Controller (MVC)
- **Models** (`models/`): Data access and business logic
- **Controllers** (`endpoints/`): HTTP request handling
- **Services** (`utils/`): Business logic and integrations

#### Dependency Injection
```javascript
// Example: LLM Provider injection
const LLMProvider = require(`../utils/AiProviders/${provider}`);
const llm = new LLMProvider(apiKey, model);
```

#### Factory Pattern
```javascript
// Vector database factory
const { getVectorDbClass } = require('./utils/helpers');
const VectorDb = getVectorDbClass();
```

### Service Layer Architecture

#### AI Providers (`utils/AiProviders/`)
Unified interface for different LLM providers:

```javascript
class OpenAIProvider {
  constructor(apiKey, model) {
    this.client = new OpenAI({ apiKey });
    this.model = model;
  }

  async streamChat(messages, workspace, options = {}) {
    // Implementation
  }
}
```

#### Embedding Engines (`utils/EmbeddingEngines/`)
Text embedding providers for vector search:

```javascript
class NativeEmbedder {
  async embed(textChunks) {
    // Local embedding using Transformers.js
  }
}
```

#### Vector Databases (`utils/vectorDbProviders/`)
Vector storage and similarity search:

```javascript
class LanceDbProvider {
  async addVectors(vectors, metadata) {
    // Store vectors in LanceDB
  }

  async similaritySearch(query, workspace, count = 4) {
    // Retrieve similar vectors
  }
}
```

### Key Utilities

#### Document Processing (`utils/DocumentManager/`)
- File upload handling
- Text extraction from PDFs, documents
- Text chunking for embeddings
- Metadata extraction

#### Chat Processing (`utils/chats/`)
- Message streaming
- Context assembly
- Response formatting
- Chat history management

#### Background Jobs (`jobs/`)
- Document synchronization
- Vector database maintenance
- Telemetry reporting

### Error Handling Patterns

#### Service Layer Errors
```javascript
async function someOperation() {
  try {
    const result = await riskyOperation();
    return { result, error: null };
  } catch (error) {
    console.error(error.message);
    return { result: null, error: error.message };
  }
}
```

#### Controller Layer Errors
```javascript
async function endpoint(req, res) {
  try {
    const { result, error } = await service.operation();
    if (error) {
      return res.status(400).json({ error });
    }
    res.json({ result });
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: "Internal server error" });
  }
}
```

## Production Deployment

### Environment Configuration
```env
NODE_ENV=production
SERVER_PORT=3001
HTTPS_CERT_PATH=/path/to/cert.pem
HTTPS_KEY_PATH=/path/to/key.pem
```

### Process Management
```bash
# Using PM2
pm2 start index.js --name anythingllm-server

# Using systemd
sudo systemctl enable anythingllm
sudo systemctl start anythingllm
```

### Docker Deployment
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3001
CMD ["node", "index.js"]
```

### Nginx Reverse Proxy
```nginx
server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## Security Considerations

- **API Keys**: Store securely, rotate regularly
- **JWT Secrets**: Use strong, unique secrets
- **File Uploads**: Validate file types and sizes
- **SQL Injection**: Prevented by Prisma ORM
- **XSS Protection**: Content Security Policy headers
- **Rate Limiting**: Implement for API endpoints
- **HTTPS**: Required for production deployments

## Performance Optimization

- **Database Indexing**: Optimized queries via Prisma
- **Vector Caching**: Embeddings cached to reduce computation
- **Connection Pooling**: Database connection management
- **Streaming Responses**: Real-time chat via Server-Sent Events
- **File Compression**: Gzip compression for responses
- **CDN Integration**: Static asset optimization

## Contributing

1. Follow existing code patterns and conventions
2. Add tests for new functionality
3. Update documentation for API changes
4. Run linting: `npm run lint`
5. Test with both single-user and multi-user modes

## License

MIT License - see LICENSE file for details.

---

**Note**: This server is part of the AnythingLLM project. For frontend setup and full deployment instructions, see the main project repository.