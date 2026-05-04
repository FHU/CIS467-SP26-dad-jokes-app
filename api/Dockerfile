FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json  ./

#---- Dev stage---
FROM base AS dev
RUN npm install
COPY . .
CMD [ "npm", "run", "dev"]


# ---- Build Stage ----
FROM base AS build 
RUN npm ci
COPY . . 
RUN npm run build


# ---- Production Stage ----
FROM node:20-alpine  AS prod
WORKDIR /app   

COPY package*.json  ./
RUN npm ci --omit=dev
# Copy compiled JS from builder
COPY --from=build /app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/index.js"]