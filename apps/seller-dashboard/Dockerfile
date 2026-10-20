# Build with the repository root as context:
# docker build -f apps/seller-dashboard/Dockerfile -t creatorplaza-seller-dashboard .
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY apps/seller-dashboard/package.json apps/seller-dashboard/pnpm-lock.yaml apps/seller-dashboard/pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile
COPY apps/seller-dashboard/ ./
RUN VITE_BASE_PATH=/sellers/ pnpm run build

FROM nginxinc/nginx-unprivileged:stable-alpine
COPY apps/seller-dashboard/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build --chown=nginx:nginx /app/dist /usr/share/nginx/html/sellers
EXPOSE 8080
