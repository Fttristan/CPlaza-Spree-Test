# Build with the repository root as context:
# docker build -f apps/dashboard/Dockerfile -t creatorplaza-dashboard .
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY apps/dashboard/package.json apps/dashboard/pnpm-lock.yaml apps/dashboard/pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile
COPY apps/dashboard/ ./
RUN VITE_BASE_PATH=/dashboard/ pnpm run build

FROM nginxinc/nginx-unprivileged:stable-alpine
COPY apps/dashboard/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build --chown=nginx:nginx /app/dist /usr/share/nginx/html/dashboard
EXPOSE 8080
