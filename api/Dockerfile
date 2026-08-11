# syntax=docker/dockerfile:1

FROM node:22-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS dev
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY src ./src
USER node
CMD ["npm", "run", "dev"]

FROM node:22-alpine AS prod
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY package.json ./
COPY src ./src

USER node
ENV NODE_ENV=production

EXPOSE 8080
CMD ["node", "src/server.js"]
