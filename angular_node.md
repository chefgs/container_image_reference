# Angular & Node Container Image Reference

This docs contains information for one for your Angular UI frontend and one for your Node.js backend. This separation allows you to develop, build, and deploy them independently, which provides more flexibility and better scalability.

## Dockerfile for Angular UI (Frontend)

```dockerfile
# Frontend Dockerfile
FROM node:18-alpine AS base
WORKDIR /app

# Install pnpm globally
RUN npm install -g pnpm

# Dependencies stage
FROM base AS dependencies
COPY package.json pnpm-lock.yaml* ./
RUN pnpm install --frozen-lockfile

# Build stage
FROM dependencies AS build
COPY . .
# Set production environment for Angular build
ENV NODE_ENV production
RUN pnpm run build

# Production stage - use nginx to serve static files efficiently
FROM nginx:alpine AS production
# Copy nginx configuration if you have a custom one
# COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy built Angular files to nginx serve directory
COPY --from=build /app/dist/angular-app /usr/share/nginx/html

# Expose the port for the frontend
EXPOSE 80

# No need for custom CMD as nginx will start automatically
```

## Dockerfile for Node.js Backend

```dockerfile
# Backend Dockerfile
FROM node:18-alpine AS base
WORKDIR /app

# Install pnpm globally
RUN npm install -g pnpm

# Dependencies stage
FROM base AS dependencies
COPY package.json pnpm-lock.yaml* ./
RUN pnpm install --frozen-lockfile

# Build stage
FROM dependencies AS build
COPY . .
ENV NODE_ENV production
RUN pnpm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app

# Set to production environment
ENV NODE_ENV production

# Create non-root user for security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

# Copy package.json and built files
COPY --from=build /app/package.json ./
COPY --from=build /app/dist ./dist

# Copy production node_modules
COPY --from=dependencies /app/node_modules ./node_modules
RUN pnpm prune --prod

# Set proper permissions
RUN mkdir -p /app/logs
RUN chown -R appuser:nodejs /app

# Switch to non-root user
USER appuser

# Expose the port for the backend API
EXPOSE 3000

# Start the backend application
CMD ["node", "dist/main.js"]
```

## Additional Configuration Files

### Create `.dockerignore` for UI:

```
node_modules
npm-debug.log
yarn-debug.log
yarn-error.log
pnpm-debug.log
.git
.github
.vscode
coverage
dist
e2e
*.md
.env*
```

### Create `.dockerignore` for Backend:

```
node_modules
npm-debug.log
yarn-debug.log
yarn-error.log
pnpm-debug.log
.git
.github
.vscode
coverage
*.md
.env*
logs
```

## Notes on Usage

1. **Directory Structure**:
   - These Dockerfiles should be placed in the respective frontend and backend directories
   - Adjust the build and output paths based on your specific project structure

2. **Building the Images**:
   ```bash
   # Build UI
   cd frontend
   docker build -t my-angular-app:latest .

   # Build Backend
   cd ../backend
   docker build -t my-node-backend:latest .
   ```

3. **Running the Containers**:
   ```bash
   # Run UI
   docker run -d -p 80:80 my-angular-app:latest

   # Run Backend
   docker run -d -p 3000:3000 my-node-backend:latest
   ```

4. **Environment Variables**:
   - For the backend, you may want to add environment variables:
     ```bash
     docker run -d -p 3000:3000 -e DB_CONNECTION=xxx my-node-backend:latest
     ```

These Dockerfiles are optimized for production use with a focus on security, performance, and minimal image size.

---

Comprehensive Dockerfile for Angular UI and Node.js Backend scenario.

# Dockerfile for Angular UI and Node.js Backend

Here's a production-ready Dockerfile that handles both Angular UI build and Node.js backend:

```dockerfile
# Base stage for shared dependencies
FROM node:18-alpine AS base
RUN npm install -g pnpm
WORKDIR /app

# Frontend (Angular) build stage
FROM base AS frontend-builder
WORKDIR /app/frontend

# Copy frontend package files
COPY frontend/package.json frontend/pnpm-lock.yaml* ./
RUN pnpm install --frozen-lockfile

# Copy frontend source code and build
COPY frontend/ ./
RUN pnpm run build --prod

# Backend dependencies stage
FROM base AS backend-deps
WORKDIR /app/backend

# Copy backend package files
COPY backend/package.json backend/pnpm-lock.yaml* ./
RUN pnpm install --frozen-lockfile

# Backend build stage
FROM base AS backend-builder
WORKDIR /app/backend

# Copy backend dependencies
COPY --from=backend-deps /app/backend/node_modules ./node_modules

# Copy backend source code and build
COPY backend/ ./
RUN pnpm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app

# Set production environment
ENV NODE_ENV production

# Create non-root user for security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

# Copy backend built artifacts and dependencies
COPY --from=backend-builder /app/backend/dist ./dist
COPY --from=backend-builder /app/backend/package.json ./
COPY --from=backend-deps /app/backend/node_modules ./node_modules

# Remove development dependencies to reduce image size
RUN pnpm prune --prod

# Copy frontend built artifacts to be served by backend
COPY --from=frontend-builder /app/frontend/dist/angular-app ./public

# Create appropriate directories with proper permissions
RUN mkdir -p /app/logs
RUN chown -R appuser:nodejs /app

# Switch to non-root user
USER appuser

# Expose the port the app will run on
EXPOSE 3000

# Start the application
CMD ["node", "dist/main.js"]
```

## Key Features of this Dockerfile:

1. **Multi-stage build** to keep the final image small
2. **Separated frontend and backend builds** for better caching and modularity
3. **pnpm** for efficient dependency management
4. **Production optimization** with pruning of dev dependencies
5. **Security best practices** with non-root user
6. **Properly structured** to serve Angular frontend from Node.js backend

## Important Considerations:

1. **Directory Structure Assumption**:
   - Frontend code in `/frontend` directory
   - Backend code in `/backend` directory

2. **Build Scripts**:
   - Assumes Angular build outputs to `dist/angular-app`
   - Assumes Node.js build outputs to `dist/main.js`

3. **Configuration**:
   - You may need to adjust paths based on your project structure
   - Environment variables should be configured in your deployment system

4. **File Serving**:
   - Assumes backend serves static files from public directory

5. **Docker Build Command**:
   ```bash
   docker build -t myapp:production .
   ```

You may need to adjust paths and commands based on your specific Angular and Node.js application structure.
