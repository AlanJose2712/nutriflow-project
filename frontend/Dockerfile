# ---------- Stage 1: build the static site ----------
FROM node:22-alpine AS build
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# Vite bakes this value into the JS files at build time
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL

RUN npm run build

# ---------- Stage 2: serve with nginx (no Node in final image) ----------
FROM nginxinc/nginx-unprivileged:alpine

# Patch OS packages (fixes brotli-libs CVE), then drop back to non-root
USER root
RUN apk upgrade --no-cache
USER 101

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 8080
