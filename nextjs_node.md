# NextJS and Node Dockerfile

Dockerfile that uses pnpm throughout the build process for your Next.js application. Using pnpm can provide faster installation and more efficient disk space usage compared to npm or yarn.

Here's an optimized Dockerfile using pnpm:

```dockerfile
# Base stage for dependencies and build
FROM node:18-alpine AS base

# Set working directory
WORKDIR /app

# Install pnpm globally
RUN npm install -g pnpm

# Install dependencies stage
FROM base AS deps
# Add libc6-compat for Node.js Alpine compatibility
RUN apk add --no-cache libc6-compat

# Copy package files
COPY package.json pnpm-lock.yaml* ./

# Install dependencies with frozen lockfile for reproducible builds
RUN pnpm install --frozen-lockfile

# Build stage
FROM base AS builder

# Copy dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Disable Next.js telemetry for privacy
ENV NEXT_TELEMETRY_DISABLED 1

# Build the application
RUN pnpm build

# Optional: Remove dev dependencies to reduce image size
RUN pnpm prune --prod

# Production stage
FROM node:18-alpine AS runner
WORKDIR /app

# Set to production environment
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

# Add a non-root user for security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy only necessary files from builder
COPY --from=builder /app/public ./public
COPY --from=builder /app/next.config.js ./
COPY --from=builder /app/package.json ./

# Copy the build output, excluding cache
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/.next/server ./.next/server
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules

# Switch to non-root user
USER nextjs

# Expose the port the app will run on
EXPOSE 3000

# Start the application
CMD ["node_modules/.bin/next", "start"]
```

To make this work optimally with your Next.js configuration:

1. Make sure your next.config.js is properly set up (which it appears to be)
2. Create a `.dockerignore` file with the following content to exclude unnecessary files:

```
node_modules
.next/cache
.git
.github
.vscode
```

This Dockerfile:
1. Uses pnpm for all dependency management
2. Implements a multi-stage build process to minimize the final image size
3. Properly handles dependencies with pnpm's frozen lockfile feature
4. Runs the prune command to remove development dependencies
5. Sets up security best practices like running as a non-root user
6. Excludes the cache directory to keep the final image smaller

Let me know if you'd like me to make any specific adjustments to this Dockerfile!

---

Dockerfile for Next.js application that follows best practices for production deployment. This Dockerfile will ensure your application is built efficiently and optimized for production use.

```dockerfile
# Base stage for dependencies and build
FROM node:18-alpine AS base

# Set working directory
WORKDIR /app

# Install pnpm globally
RUN npm install -g pnpm

# Install dependencies stage
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine
# to understand why libc6-compat might be needed
RUN apk add --no-cache libc6-compat

# Copy package.json and related files
COPY package.json pnpm-lock.yaml* ./

# Install dependencies with frozen lockfile to ensure deterministic builds
RUN pnpm install --frozen-lockfile

# Build stage
FROM base AS builder

# Copy dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects anonymous telemetry data - disable if desired
ENV NEXT_TELEMETRY_DISABLED 1

# Build the application with production environment
RUN pnpm build

# Production stage - creating a smaller image for deployment
FROM node:18-alpine AS runner

WORKDIR /app

# Set to production environment
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

# Add a non-root user for security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy necessary files from builder
COPY --from=builder /app/public ./public
COPY --from=builder /app/next.config.js ./

# Important: Specifically exclude the cache directory to reduce image size
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./

# Switch to non-root user
USER nextjs

# Expose the port the app will run on
EXPOSE 3000

# Set the correct environment variable for the port
ENV PORT 3000

# Start the application
CMD ["node", "server.js"]
```

This Dockerfile:

1. Uses a multi-stage build process to keep the final image small
2. Installs dependencies with pnpm using frozen lockfile for reproducible builds
3. Properly handles the Next.js build process
4. Creates a production-optimized image that excludes development dependencies
5. Implements security best practices like running with a non-root user
6. Excludes the cache directory to keep the image size smaller
7. Uses the standalone output feature of Next.js for better performance

To use this Dockerfile, make sure you've set `output: 'standalone'` in your next.config.js file as follows:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  swcMinify: true,
  output: 'standalone', // Add this line for standalone output
  compress: true,       // Add compression for better performance
  images: {
    unoptimized: false,
    domains: [],
    formats: ['image/avif', 'image/webp'],
  },
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
    GOOGLE_AI_API_KEY: process.env.GOOGLE_AI_API_KEY,
    MONGODB_URI: process.env.MONGODB_URI,
  }
}

const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

module.exports = withBundleAnalyzer(nextConfig)
```

With this Dockerfile and updated configuration, you'll have an optimized production deployment for your Next.js application.
