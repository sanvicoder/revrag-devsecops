# Dockerfile Audit

## 1. Mutable Base Image — `node:latest`

### Problem
The original Dockerfile used the mutable node:latest tag. This means the Node.js version and underlying operating-system packages could change without a corresponding change to the Dockerfile, reducing build reproducibility and making security auditing more difficult.

### Fix
I changed the base image to node:22-bookworm-slim. This pins the major Node.js version and uses a smaller Debian-based variant, making the build more predictable and reducing unnecessary packages. For even stronger reproducibility, the image could additionally be pinned to a verified digest after the application has been tested against the selected image.

## 2. Uncontrolled Build Context — `COPY . .`

### Problem
The original Dockerfile used COPY . ., which copies the Docker build context into the image. Without exclusions, this could include files such as environment files, credentials, Git metadata, logs, or local development artifacts.

### Fix
I added a .dockerignore file that excludes .git, .github, environment files, logs, node_modules, and other files that should not be included in the production image. This reduces the risk of accidentally copying sensitive or unnecessary files into the container.

## 3. Non-Reproducible Dependency Installation — `npm install`

### Problem
The original Dockerfile uses npm install. Without a committed package-lock.json, dependency resolution may change over time, which can make builds less reproducible.

### Fix
A production project should commit its package-lock.json and use npm ci to install the exact locked dependency versions. Because the supplied assignment repository does not contain the Node.js application's dependency files, I kept npm install rather than introducing npm ci without a lockfile.

## 4. Hardcoded Application Secret — `SECRET_KEY`

### Problem
The original Dockerfile embedded SECRET_KEY directly into the image. Secrets should not be stored in Dockerfiles or container images because they may be exposed through source control, image layers, or anyone with access to the image.

### Fix
I removed the hardcoded SECRET_KEY from the Dockerfile. The application secret should instead be supplied at runtime through GitHub Secrets, a container secret mechanism, or a dedicated secret-management service.

## 5. Hardcoded Database Password — `DB_PASSWORD`

### Problem
The original Dockerfile embedded a database password directly into the image. The supplied password was also a weak static credential, so it should be treated as compromised.

### Fix
I removed DB_PASSWORD from the Dockerfile. The credential should be supplied securely at runtime, and the original exposed password should be rotated rather than reused.

## 6. Unnecessary Runtime Packages — `curl`, `vim`, and `wget`

### Problem
The original Dockerfile installed curl, vim, and wget, but there was no evidence that the application required these tools at runtime. Unnecessary packages increase image size, maintenance burden, and potential vulnerability surface.

### Fix
I removed the APT installation step because these tools are not required by the Node.js application. The resulting image contains fewer unnecessary packages and has a smaller attack surface.

## 7. Unnecessary SSH Port — `EXPOSE 22`

### Problem
The original Dockerfile exposed port 22 even though the container runs a Node.js application and does not start an SSH server. Exposing an unnecessary port adds confusing configuration and can encourage inappropriate SSH access to application containers.

### Fix
I removed EXPOSE 22 and kept only EXPOSE 3000, which is the port used by the Node.js application.

## 8. Container Runs as Root

### Problem
The original Dockerfile did not specify a non-root user, so the application would run as root inside the container. If the application were compromised, unnecessary root privileges could increase the potential impact.

### Fix
I added USER node so the application runs as the non-root node user provided by the official Node.js image. This follows the principle of least privilege and limits what an attacker could do inside the container.

## 9. APT Package Metadata / Image Hygiene

### Problem
The original Dockerfile installed Debian packages with apt-get but did not clean the APT package metadata afterward. If OS packages are installed, leaving the package lists in the image adds unnecessary data and increases image size.

### Fix
I removed the unnecessary APT installation entirely because curl, vim, and wget are not required by the application. If OS packages are required in the future, the package installation and cleanup should be performed in the same Docker layer.