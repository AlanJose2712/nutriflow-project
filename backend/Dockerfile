FROM node:22-slim

ENV NODE_ENV=production
WORKDIR /app

# Copy only dependency files first so Docker can cache this layer
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Copy the application code
COPY --chown=node:node . .

# Run as the non-root "node" user (built into the base image)
USER node

EXPOSE 8000
CMD ["node", "index.js"]
