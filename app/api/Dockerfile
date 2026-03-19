FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json tsconfig.json ./
RUN npm ci
COPY src/ src/
RUN npm run build

FROM node:22-alpine
RUN apk upgrade --no-cache && apk add --no-cache "zlib>=1.3.2-r0"
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=builder /app/dist dist/
EXPOSE 3000
CMD ["node", "dist/main"]
