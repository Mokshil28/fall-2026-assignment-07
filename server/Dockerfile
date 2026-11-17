FROM node:24-alpine AS builder
WORKDIR /app
COPY package*.json ./
# Compilation only uses TypeScript; skip unrelated development install scripts.
RUN npm ci --ignore-scripts --omit=optional --no-audit --no-fund
COPY tsconfig*.json ./
COPY src ./src
RUN npm run build

FROM node:24-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=builder --chown=node:node /app/dist ./dist
USER node
EXPOSE 8080
CMD ["node", "dist/index.js"]
