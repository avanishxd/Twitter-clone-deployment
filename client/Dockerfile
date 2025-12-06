# ====== BUILD STAGE ======
FROM node:18 AS build

WORKDIR /app

# Install deps
COPY package*.json ./
RUN npm install --legacy-peer-deps

# Copy source and build
COPY . .
RUN npm run build

# ====== RUNTIME STAGE ======
FROM nginx:alpine

# Copy build artifacts from previous stage
COPY --from=build /app/build /usr/share/nginx/html

# Expose HTTP
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
