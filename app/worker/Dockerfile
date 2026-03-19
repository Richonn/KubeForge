FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json tsconfig.json ./
RUN npm install
COPY src/ src/
RUN npm run build

FROM node:22-alpine
RUN apk upgrade --no-cache && \
    apk add --no-cache "zlib>=1.3.2-r0" && \
    rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx
WORKDIR /app
COPY --from=builder /app/dist dist/
CMD ["node", "dist/index.js"]
