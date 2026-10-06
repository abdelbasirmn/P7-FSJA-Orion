# Build this file from the repository root:
# docker build -f front/Dockerfile -t orion-front:local .
FROM node:20-alpine AS build
WORKDIR /src
COPY front/package*.json ./
RUN npm ci
COPY front/ .
RUN npm run build -- --configuration production

FROM caddy:2-alpine
WORKDIR /app
COPY --from=build /src/dist/microcrm/browser /app/front
COPY misc/docker/Caddyfile /app/Caddyfile
EXPOSE 80 443
CMD ["caddy","run","--config","/app/Caddyfile"]
