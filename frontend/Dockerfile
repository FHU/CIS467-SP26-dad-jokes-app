FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json  ./
RUN npm install
COPY . .

# --- dev stage----
FROM base AS dev 

CMD [ "npm", "run", "dev"]




# ---- Build Stage ----
FROM base AS build 

RUN npm install


# ---- Production Stage ----
FROM nginx:alpine 

# Copy built React assets
COPY --from=build /app/dist /usr/share/nginx/html
# Copy custom nginx config (handles SPA routing + API proxy)
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]