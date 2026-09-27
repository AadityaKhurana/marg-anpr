# syntax=docker/dockerfile:1
# Production frontend image: build the Vite bundle, serve it with nginx.
# nginx also reverse-proxies /api and /ws to the api container, so the whole
# app stays on one origin (no CORS, no separate API port exposed).

# ---- build stage ----
FROM node:20-slim AS build
WORKDIR /app

# Public build-time config. Vite bakes VITE_* into the bundle at build time.
# Defaults target the live Dwarka demo; override via compose build args if needed.
ARG VITE_USE_MOCK=false
ARG VITE_API_BASE_URL=/api
ARG VITE_WS_URL=/ws/live
ARG VITE_MAP_CENTER_LAT=28.5900
ARG VITE_MAP_CENTER_LNG=77.0470
ARG VITE_MAP_ZOOM=13
# Optional keyed basemap URL. Leave empty to use the app's keyless Carto voyager
# fallback (config.ts uses `||`, so an empty string still falls back cleanly).
# To use a keyed provider, pass the full tile URL (including the key/token).
ARG VITE_CARTO_BASEMAP_URL=
ENV VITE_USE_MOCK=$VITE_USE_MOCK \
    VITE_API_BASE_URL=$VITE_API_BASE_URL \
    VITE_WS_URL=$VITE_WS_URL \
    VITE_MAP_CENTER_LAT=$VITE_MAP_CENTER_LAT \
    VITE_MAP_CENTER_LNG=$VITE_MAP_CENTER_LNG \
    VITE_MAP_ZOOM=$VITE_MAP_ZOOM \
    VITE_CARTO_BASEMAP_URL=$VITE_CARTO_BASEMAP_URL

COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- serve stage ----
FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
