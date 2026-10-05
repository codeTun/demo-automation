# Stage 1: compile the Angular application.
# Nothing has to be installed on the Jenkins machine: the build runs inside Docker.
FROM node:22-slim AS build
WORKDIR /build
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY . .
RUN npx ng build

# Stage 2: serve the compiled files with nginx.
# The Debian-based image, not the Alpine one: on this cluster, nginx in the Alpine image
# could not resolve the name of the API gateway when it started.
FROM nginx:stable
RUN rm -rf /usr/share/nginx/html/*
COPY --from=build /build/dist/fleetman-webapp/browser/ /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
