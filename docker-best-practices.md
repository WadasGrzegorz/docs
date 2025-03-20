# Docker Best Practices for Node.js Applications

Docker is a powerful tool for containerizing applications and when using it with Node.js, following best practices ensures optimal performance, security and maintainability. Below are some key best practices with examples to help you build efficient and secure Node.js applications using Docker.

### 1. Use a Minimal and Official Base Image

Choosing a lightweight base image reduces the attack surface and improves performance. The official Node.js images provide optimized environments. Which provide you better security, performance, compressed image size and maintainability.

```Dockerfile
# Use official Node.js image https://hub.docker.com/_/node
FROM node:22-alpine
# rest of your docker file
```

### 2. Leverage Multi-Stage Builds

Multi-stage builds help keep your final image clean and small by separating the build and runtime environments. This reduces the final image size and improves security. Less packages means less vulnerabilities.

```Dockerfile
# stage 1 - build your application
FROM node:22-alpine AS builder

# prepare your build
COPY package.json yarn.lock ./
RUN yarn install
COPY . .
# build your application
RUN yarn build
# remove development dependencies
RUN yarn install --production

# stage 2 - run your application - only production dependency needed
FROM node:22-alpine AS runner

COPY --from=builder / .
COPY --from=builder /node_modules ./node_modules

CMD ["node", "server.js"]
EXPOSE 3000
```

### 3. Use .dockerignore to Exclude Unnecessary Files

Avoid copying unnecessary files into your Docker image to keep it lightweight.

```dockerignore
node_modules
npm-debug.log
dist
.DS_Store
.git
.env
```

### 4. Run the Container as a Non-Root User

Make sure that your Node.js application runs as a non-root user to improve security. This reduces the impact of security vulnerabilities. Even if you need some root access, remember to switch back to a non-root user.
This will allow you to follow principle of least privilege.

```Dockerfile
FROM node:22-alpine

USER root
# some commands which needs root access
RUN touch /app/test.txt
# back to non-root user
USER node
# COPY will use node user privileges
COPY package.json yarn.lock ./
# remove development dependencies
RUN yarn install --production
COPY . .
CMD ["node", "server.js"]
EXPOSE 3000
```

### 5. Optimize Caching in Docker Layers
Order of your command included in your Dockerfile is important. Docker caches layers, so make sure to order your commands from least to most frequently changing.
For example ordering COPY and RUN commands efficiently ensures Docker can reuse cached layers and speed up builds.

Bad example:
```Dockerfile
COPY . .
# this step will never use cache, if any file will be changed
RUN yarn install
```
Good example:
```Dockerfile
COPY package.json yarn.lock ./
# this step will use cache until package.json or yarn.lock will be changed (package will be updated)
RUN yarn install
COPY . .
```
Such approach will help you to avoid unnecessary package installation on every change in your code which ensures faster builds and lower image size.
Read more about cache here: https://docs.docker.com/build/cache/optimize/

### 6. Use Environment Variables for Configuration
Do not hardcode configuration values in your Dockerfile or application code. Use environment variables to pass configuration values to your application. This makes your application more flexible and secure.

```Dockerfile
ARG NODE_ENV=production
ENV NODE_ENV=$NODE_ENV
```
and then build your image with:
```shell
docker build --build-arg NODE_ENV=development -t myapp .
```

### 7. Use Docker Secrets for Sensitive Data
Avoid storing sensitive information such as API keys, database passwords or credentials in environment variables or the Docker image. Instead, use Docker secrets.

```Dockerfile
# Secret is ephemeral, so we need to set it as an environment variable to satisfy yarn build
ENV SOME_SECRET_TOKEN=""

# Getting secret from the secret store - is available only in this line (layer)
RUN --mount=type=secret,id=SOME_SECRET_TOKEN,env=SOME_SECRET_TOKEN \
    yarn install
```
then you can run your container with:
```shell
docker run --secret id=SOME_SECRET_TOKEN myapp
```
More about build secrets you can find [here](https://docs.docker.com/build/building/secrets/)

### 8. Use ephemeral containers for your application
Make sure that your application is stateless and does not store any data in the container. This means that you can destroy your container at any time without losing any data. This will allow you to scale your application horizontally and make it more resilient.

### 9. Scan your Docker images for vulnerabilities
Use tools like [Trivy](https://trivy.dev/latest/) to scan your Docker images for vulnerabilities. This will help you identify and fix security issues before deploying your application.

```shell
trivy image --severity CRITICAL,HIGH --exit-code 1 --ignore-unfixed your-image:tag
```

### 10. Keep your Docker images up to date
Regularly update your Docker images and dependencies to ensure that you are using the latest security patches and bug fixes. This will help you keep your application secure and up to date. You can use dependabot to automate this process.

```yaml
version: 2
updates:
  - package-ecosystem: 'docker'
    directory: '/'
    schedule:
      interval: 'weekly'
      day: 'thursday'
    commit-message:
      prefix: 'build(docker)'
```

Additionally, you can update you base image with:
```Dockerfile
RUN apk upgrade --no-cache
```

### 11. Choose the Right Base Image for Your Use Case
Not all applications require the same base image. Using the right image for your specific use case can improve performance and security.

Example:
If your application serves static files or runs a frontend, consider using Nginx instead of Node.js for serving content.

```Dockerfile
FROM nginx:alpine
COPY build /usr/share/nginx/html
CMD ["nginx", "-g", "daemon off;"]
```

### Conclusion

Following these best practices ensures your Node.js applications run efficiently and securely inside Docker containers. By using minimal images, multi-stage builds, caching strategies, running as a non-root user, securely managing secrets, passing environment variables via build arguments and selecting the right base image for your use case, you create a more robust and scalable environment.

Below are two examples of full Dockerfile for Node.js application:

Docker file for Nestjs/Express application:

```Dockerfile
ARG NODE_VERSION=22.14.0

FROM node:${NODE_VERSION}-alpine AS builder

WORKDIR /usr/src/app

# Secret is ephemeral, so we need to set it as an environment variable to satisfy yarn build
ENV SOME_SECRET=""

COPY --chown=node:node package.json yarn.lock ./

# Getting secret - is available only in this line (layer)
RUN --mount=type=secret,id=SOME_SECRET,env=SOME_SECRET \
    yarn install --frozen-lockfile

COPY . .

RUN yarn build
# remove development dependencies
RUN yarn install --production

FROM node:${NODE_VERSION}-alpine AS runner

RUN apk upgrade --no-cache
RUN apk add --no-cache curl bash

ARG NODE_ENV=production

ENV NODE_ENV=${NODE_ENV}

WORKDIR /usr/src/app

COPY --from=builder --chown=node:node /usr/src/app/package.json ./
COPY --from=builder --chown=node:node /usr/src/app/yarn.lock ./
COPY --from=builder --chown=node:node /usr/src/app/dist ./dist
COPY --from=builder --chown=node:node /usr/src/app/node_modules ./node_modules

USER node

CMD ["node", "dist/main"]
```

Dockerfile for React application:

```Dockerfile
# build environment

ARG NODE_VERSION=22.14.0

FROM node:${NODE_VERSION}-alpine AS builder

WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install --frozen-lock-file

COPY . .

RUN yarn build

# production environment
FROM nginx/nginx-unprivileged:1.27.4-alpine AS runner

WORKDIR /app

# optional: copy custom nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# exaple of commands which need root access
USER root
RUN apk update && apk upgrade
RUN apk add --no-cache bash curl

COPY --from=builder /app/dist /usr/share/nginx/html

# remove root access
USER nginx

```
