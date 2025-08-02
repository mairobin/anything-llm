# AnythingLLM Development Setup

This guide covers how to set up AnythingLLM for development and make code changes safely.

## Development vs Production

This project supports both **development** and **production** deployment modes:

- **Development**: Hot-reload enabled, separate processes for frontend/server/collector
- **Production**: Built frontend served by server, optimized for deployment

## Initial Setup

### Prerequisites
- Node.js v18+
- Yarn package manager

### Quick Start

1. **Clone and setup dependencies**
   ```bash
   git clone git@github.com:Mintplex-Labs/anything-llm.git
   cd anything-llm
   yarn setup
   ```

2. **Create development branch** (recommended for customizations)
   ```bash
   git checkout -b devel  # or your preferred branch name
   ```

## Development Mode


**Env Files**
Here's a comprehensive overview of all environment variables:

  Core Configuration Files:

  - server/.env.example - Server configuration (347 lines)
  - frontend/.env.example - Frontend configuration (4 lines)
  - collector/.env.example - Collector configuration (2 lines)
  - docker/.env.example - Docker configuration (350 lines)


For development with hot-reload and easy debugging:

1. **Run all services in separate terminals:**
   ```bash
   # Terminal 1 - Server
   yarn dev:server
   
   # Terminal 2 - Collector  
   yarn dev:collector
   
   # Terminal 3 - Frontend
   yarn dev:frontend
   ```

2. **Access the application:**
   - Frontend: http://localhost:3000
   - Server API: http://localhost:3001/api
   - Collector: http://localhost:8888

## Production Mode

For testing production builds or deployment:

### Environment Configuration

1. **Server Environment** (`server/.env`):
   ```bash
   cp server/.env.example server/.env
   ```
   
   **Required settings:**
   ```env
   STORAGE_DIR="/absolute/path/to/server/storage"
   ```

2. **Frontend Environment** (`frontend/.env`):
   ```env
   # For production deployment
   VITE_API_BASE='/api'
   
   # For development  
   # VITE_API_BASE='http://localhost:3001/api'
   ```

3. **Collector Environment** (`collector/.env`):
   ```bash
   cp collector/.env.example collector/.env
   ```
   
   **Required settings:**
   ```env
   STORAGE_DIR="/absolute/path/to/server/storage"
   ```

### Build and Run Production

```bash
# 1. Build frontend
cd frontend && yarn build

# 2. Copy built files to server
cp -R frontend/dist server/public

# 3. Run database migrations
cd server && npx prisma generate --schema=./prisma/schema.prisma
cd server && npx prisma migrate deploy --schema=./prisma/schema.prisma

# 4. Start services
cd server && NODE_ENV=production node index.js &
cd collector && NODE_ENV=production node index.js &
```

**Access:** http://localhost:3001

## Making Code Changes

### File Structure
```
anything-llm/
├── frontend/          # React frontend application
├── server/           # Express backend server  
├── collector/        # Document processing service
└── docker/          # Docker configurations
```

### Development Workflow

1. **Make changes** to any component:
   - Frontend: `frontend/src/` - React components, pages, utils
   - Server: `server/` - API routes, models, utilities
   - Collector: `collector/` - Document processing logic

2. **Hot reload** automatically applies changes in development mode

3. **Test changes:**
   ```bash
   # Run tests (if available)
   yarn test
   
   # Lint code
   yarn lint
   
   # Type check
   yarn typecheck
   ```

### Database Changes

If you modify the database schema:

```bash
cd server
npx prisma migrate dev --name your-migration-name
npx prisma generate
```

### Adding Dependencies

```bash
# Frontend dependencies
cd frontend && yarn add package-name

# Server dependencies  
cd server && yarn add package-name

# Collector dependencies
cd collector && yarn add package-name
```

## Preserving Customizations

### Branch Strategy
- Keep customizations on a development branch (e.g., `devel`)
- Don't make changes directly to `master` branch
- This prevents conflicts when pulling upstream updates

### Updating from Upstream
```bash
# Save your changes
git add . && git commit -m "Save current changes"

# Fetch latest changes
git fetch origin master

# Merge or rebase your changes
git merge origin/master  # or git rebase origin/master
```

### Production Deployment of Custom Changes

Follow the production setup steps above, but your customizations will be included in the build.

**⚠️ Important Notes:**
- The example update script in `BARE_METAL.md` uses `git checkout .` which **overwrites local changes**
- Always work on a separate branch to preserve your customizations
- Custom deployments are **not officially supported** by the core team

## Troubleshooting

### Common Issues

1. **STORAGE_DIR path errors**
   - Ensure absolute paths in all `.env` files
   - Create the storage directory if it doesn't exist

2. **Port conflicts**
   - Default ports: 3000 (frontend dev), 3001 (server), 8888 (collector)
   - Change ports in respective configuration files if needed

3. **Database migration errors**
   - Ensure `server/.env` is properly configured
   - Run `npx prisma generate` before migrations

4. **Frontend build issues**
   - Check `frontend/.env` has correct `VITE_API_BASE` setting
   - Clear `node_modules` and reinstall if needed: `rm -rf node_modules && yarn`

### Getting Help

- Review logs in terminal outputs
- Check the official documentation
- Remember: custom/bare-metal deployments are not officially supported

## Useful Commands

```bash
# Kill all node processes (useful for cleanup)
pkill node

# Check what's running on ports
lsof -i :3001  # server
lsof -i :8888  # collector  
lsof -i :3000  # frontend dev

# Full clean reinstall
rm -rf node_modules frontend/node_modules server/node_modules collector/node_modules
yarn setup
```

---

**Happy coding! 🚀**
