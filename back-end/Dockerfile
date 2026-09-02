# Stage 1: Install production dependencies
FROM node:22-alpine AS dependencies

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --omit=dev


# Stage 2: Create final runtime image
FROM node:22-alpine AS runtime

WORKDIR /app

ENV NODE_ENV=production

COPY --chown=node:node --from=dependencies /app/node_modules ./node_modules

COPY --chown=node:node . .

USER node

EXPOSE 3001

CMD ["npm", "start"]