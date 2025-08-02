# Restart Commands for AnythingLLM

This document contains the commands needed to stop and start the different components of AnythingLLM after making changes.

## Development Mode

### Stop All Services
```bash
# Kill all running Node.js processes (be careful - this affects all Node processes)
pkill -f node

# Or find and kill specific processes
lsof -ti:3001 | xargs kill -9  # Kill server (port 3001)
lsof -ti:3000 | xargs kill -9  # Kill frontend (port 3000)  
lsof -ti:8888 | xargs kill -9  # Kill collector (port 8888)
```

### Start Individual Services
```bash
# Start server (backend)
cd server && yarn dev

# Start frontend (UI)
cd frontend && yarn dev

# Start collector (document processing)
cd collector && yarn dev
```

### Start All Services (Concurrent)
```bash
# From root directory - starts all services simultaneously
yarn dev:all
```

## Production Mode

### Stop Production Services
```bash
# If running as systemd services or PM2, adjust accordingly
pkill -f "node.*index.js"

# Or if using specific process management
pm2 stop all  # If using PM2
```

### Start Production Services
```bash
# Start server
cd server && yarn start

# Build and serve frontend (static files served by server)
cd frontend && yarn build

# Start collector
cd collector && yarn start
```

## Docker Mode

### Stop Docker Services
```bash
cd docker
docker-compose down
```

### Start Docker Services
```bash
cd docker
docker-compose up -d
```

### Restart Docker Services
```bash
cd docker
docker-compose restart
```

### View Docker Logs
```bash
cd docker
docker-compose logs -f
```

## Quick Development Restart

After making changes, the fastest way to restart everything:

```bash
# Kill all services
pkill -f node

# Restart all development services
yarn dev:all
```

## Component-Specific Restart

### After Server Changes
```bash
# Kill only server
lsof -ti:3001 | xargs kill -9
cd server && yarn dev
```

### After Frontend Changes
```bash
# Kill only frontend
lsof -ti:3000 | xargs kill -9
cd frontend && yarn dev
```

### After Collector Changes
```bash
# Kill only collector
lsof -ti:8888 | xargs kill -9
cd collector && yarn dev
```

## Notes

- The frontend runs on port 3000 in development
- The server runs on port 3001
- The collector runs on port 8888
- In production, the frontend is built and served by the server on port 3001
- Use `yarn setup` for initial installation of all dependencies
- Database migrations: `yarn prisma:migrate` (run from root)
- If ports are in use, check with `lsof -i :PORT_NUMBER`