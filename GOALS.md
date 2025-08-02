# AnythingLLM Customization Goals & Workflow

## Fundamental Principles for Node.js Open Source Customization

### 1. **Architecture Understanding First**
Before making any changes, understand the core architecture:
- **Frontend**: React/Vite SPA with component-based architecture
- **Server**: Express.js API with Prisma ORM for database operations
- **Collector**: Document processing microservice for ingestion
- **Database**: SQLite (default) with Prisma migrations
- **Communication**: REST APIs between frontend/server, internal APIs for collector

### 2. **Development Environment Strategy**
- **Branch Management**: Always work on feature branches, never directly on `master`
- **Environment Separation**: Use development mode for rapid iteration, production for testing deployments
- **Hot Reload Workflow**: Leverage `yarn dev:*` commands for immediate feedback
- **Dependency Management**: Understand each service has its own `package.json` and dependencies

### 3. **Code Organization Principles**

#### Frontend (`frontend/src/`)
```
pages/          # Route-level components
components/     # Reusable UI components  
utils/          # Helper functions and utilities
models/         # API client functions
hooks/          # Custom React hooks
contexts/       # Global state management
```

#### Backend (`server/`)
```
endpoints/      # API route handlers
models/         # Database models and business logic
utils/          # Shared utilities and helpers
prisma/         # Database schema and migrations
middleware/     # Express middleware functions
```

#### Collector (`collector/`)
```
utils/          # File processing utilities
processors/     # Document type handlers
index.js        # Main collector service
```

### 4. **Common Customization Patterns**

#### A. Adding New LLM Providers
1. **Server-side**: Create new provider in `server/utils/AiProviders/`
2. **Frontend**: Add provider options in `frontend/src/components/LLMSelection/`
3. **Environment**: Add provider config to `.env` template
4. **Integration**: Wire up provider selection in settings UI

#### B. Custom UI Components
1. **Component Creation**: Follow existing component patterns in `frontend/src/components/`
2. **Styling**: Use Tailwind CSS classes consistent with existing design
3. **State Management**: Leverage React hooks or context providers
4. **API Integration**: Create corresponding API calls in `frontend/src/models/`

#### C. Database Schema Changes
1. **Schema Modification**: Edit `server/prisma/schema.prisma`
2. **Migration Creation**: Run `npx prisma migrate dev --name description`
3. **Client Generation**: Run `npx prisma generate`
4. **Model Updates**: Update corresponding server models

#### D. New API Endpoints
1. **Route Creation**: Add routes in `server/endpoints/`
2. **Middleware**: Apply authentication/validation middleware
3. **Model Integration**: Connect to database via Prisma models
4. **Frontend Client**: Create API client functions in `frontend/src/models/`

### 5. **Development Workflow Best Practices**

#### Initial Setup
```bash
git checkout -b feature/your-feature-name
yarn setup                    # Install all dependencies
```

#### Development Cycle
```bash
# Terminal 1 - Server (with hot reload)
yarn dev:server

# Terminal 2 - Collector  
yarn dev:collector

# Terminal 3 - Frontend (with hot reload)
yarn dev:frontend

# Make changes, test immediately via hot reload
# Frontend: http://localhost:3000
# API: http://localhost:3001/api
```

#### Testing & Validation
```bash
# Run linting
yarn lint

# Type checking (if applicable)
yarn typecheck

# Database migrations (when schema changes)
cd server && npx prisma migrate dev
```

#### Production Testing
```bash
# Build and test production deployment
yarn build                   # Build frontend
cp -R frontend/dist server/public
NODE_ENV=production node server/index.js &
NODE_ENV=production node collector/index.js &
```

### 6. **Configuration Management**

#### Environment Variables Strategy
- **Development**: Use `.env.development` for dev-specific configs
- **Production**: Use `.env` for production deployment
- **Secrets**: Never commit API keys or sensitive data
- **Documentation**: Document all new environment variables

#### Feature Flags & Toggles
- Use environment variables for feature enablement
- Implement graceful degradation for optional features
- Create admin UI toggles for runtime configuration

### 7. **Integration Points & Extension Patterns**

#### Plugin Architecture
- **Processors**: Add new document processors in `collector/utils/`
- **Middleware**: Create reusable Express middleware in `server/middleware/`
- **Providers**: Follow provider pattern for LLMs, embeddings, vector DBs
- **Components**: Build composable React components following existing patterns

#### External Service Integration
- **API Clients**: Create dedicated client classes in `server/utils/`
- **Error Handling**: Implement consistent error patterns across services
- **Retry Logic**: Add resilience for external API calls
- **Monitoring**: Log important events for debugging

### 8. **Data Flow Understanding**

#### Request Lifecycle
1. **Frontend** → Makes API call to server
2. **Server** → Validates, processes, interacts with database
3. **Database** → Returns data via Prisma ORM
4. **Server** → Transforms data, returns JSON response
5. **Frontend** → Updates UI state with response

#### Document Processing Flow
1. **Frontend** → Uploads document
2. **Server** → Saves to storage, notifies collector
3. **Collector** → Processes document, extracts text
4. **Database** → Stores processed content and vectors
5. **Chat System** → Uses processed content for RAG

### 9. **Deployment & Maintenance Strategy**

#### Version Control
- **Feature Branches**: One branch per feature/fix
- **Commit Messages**: Clear, descriptive commit messages
- **Pull Requests**: Code review before merging
- **Tagging**: Tag stable releases

#### Update Management
- **Upstream Sync**: Regularly sync with original repository
- **Conflict Resolution**: Handle merge conflicts systematically
- **Dependency Updates**: Keep dependencies current but test thoroughly
- **Migration Scripts**: Create scripts for major updates

### 10. **Common Gotchas & Solutions**

#### Environment Issues
- **STORAGE_DIR**: Must be absolute paths in all `.env` files
- **Port Conflicts**: Ensure ports 3000, 3001, 8888 are available
- **API Base URLs**: Correct `VITE_API_BASE` setting for your deployment

#### Database Issues
- **Migration Conflicts**: Resolve schema conflicts before deploying
- **Prisma Client**: Regenerate client after schema changes
- **Data Backup**: Backup database before major changes

#### Build Issues
- **Dependency Mismatch**: Clear `node_modules` and reinstall
- **Asset Loading**: Ensure assets are correctly copied to server/public
- **Environment Variables**: Verify all required env vars are set

## Strategic Approach to Customization

### Phase 1: Understanding (Week 1-2)
- Set up development environment
- Explore codebase structure
- Trace through key user flows
- Identify customization targets

### Phase 2: Planning (Week 3)
- Define specific customization goals
- Map out required changes across services
- Plan database schema modifications
- Design new UI components

### Phase 3: Implementation (Week 4-N)
- Implement changes incrementally
- Test in development mode continuously
- Create comprehensive documentation
- Prepare production deployment

### Phase 4: Integration & Testing (Final Week)
- End-to-end testing of all features
- Production deployment testing
- Performance optimization
- User acceptance testing

## Success Metrics
- ✅ Clean, maintainable code following project patterns
- ✅ Comprehensive documentation of changes
- ✅ Successful production deployment without breaking existing features
- ✅ Easy maintenance and future updates
- ✅ Preserved ability to sync with upstream changes

---

**Remember**: The goal is not just to make it work, but to make it work *sustainably* within the existing architecture and patterns.