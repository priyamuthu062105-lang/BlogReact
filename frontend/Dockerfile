
# Stage 1: Build React frontend
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ARG VITE_API_URL=/api
ENV VITE_API_URL=${VITE_API_URL}

RUN npm run build


# Stage 2: Production Nginx server
FROM nginxinc/nginx-unprivileged:stable-alpine AS production

# Apply Alpine security package updates
USER root
RUN apk upgrade --no-cache

# Run the application as a non-root user
USER 101

# Copy React production build
COPY --from=builder /app/dist /usr/share/nginx/html

# Nginx configuration
COPY default.conf.template /etc/nginx/templates/default.conf.template

ENV BACKEND_HOST=backend
ENV BACKEND_PORT=5000

EXPOSE 8080

# Container health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -q -O /dev/null http://127.0.0.1:8080/ || exit 1
