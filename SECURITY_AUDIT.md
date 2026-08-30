# Dockerfile Audit

## 1. Mutable Base Image — `node:latest`

### Problem

The original Dockerfile used the mutable `node:latest` tag. This means the Node.js version and underlying OS packages can change without a corresponding change to the Dockerfile, reducing build reproducibility and making security auditing more difficult.

### Fix

I changed the base image to `node:22-alpine` and pinned it to a specific image digest. This makes the build reproducible and uses a smaller Alpine-based image with fewer packages and a smaller attack surface.

The final base image is:

```dockerfile
FROM node:22-alpine@sha256:c610fcdfb1d5b4740dd70c284ed3cb16bb857e0f7166196e36a5501df7a3aa32
```

---

## 2. Uncontrolled Build Context — `COPY . .`

### Problem

The original Dockerfile used `COPY . .`, which copies the Docker build context into the image. Without exclusions, this could include files such as environment files, credentials, Git metadata, logs, or local development artifacts.

### Fix

I added a `.dockerignore` file that excludes `.git`, `.github`, environment files, logs, `node_modules`, and other files that should not be included in the production image.

This reduces the risk of accidentally copying sensitive or unnecessary files into the container.

---

## 3. Non-Reproducible Dependency Installation — `npm install`

### Problem

The original Dockerfile used `npm install`. For production builds, dependency installation should use the project's lockfile to improve reproducibility.

### Fix

The project contains a `package-lock.json` using lockfile version 3, so I changed the dependency installation to:

```dockerfile
COPY package*.json ./

RUN npm ci --omit=dev
```

`npm ci` is designed for clean installations using the project's lockfile. The `--omit=dev` option prevents development dependencies from being installed in the production image, reducing unnecessary packages and attack surface.

The current `package-lock.json` contains the root package metadata but does not currently contain third-party dependency entries. Therefore, `npm ci` currently installs no application dependencies, but it establishes the appropriate production installation pattern if dependencies are added later.

---

## 4. Hardcoded Application Secret — `SECRET_KEY`

### Problem

The original Dockerfile embedded `SECRET_KEY` directly into the image.

Secrets should not be stored in Dockerfiles or container images because they may be exposed through source control, image layers, or anyone with access to the image.

### Fix

I removed the hardcoded `SECRET_KEY` from the Dockerfile.

The application secret should instead be supplied at runtime through GitHub Secrets, a container secret mechanism, or a dedicated secret-management service.

---

## 5. Hardcoded Database Password — `DB_PASSWORD`

### Problem

The original Dockerfile embedded a database password directly into the image.

The supplied password was also a weak static credential, so it should be treated as compromised.

### Fix

I removed `DB_PASSWORD` from the Dockerfile.

The credential should be supplied securely at runtime, and the original exposed password should be rotated rather than reused.

---

## 6. Unnecessary Runtime Packages — `curl`, `vim`, and `wget`

### Problem

The original Dockerfile installed `curl`, `vim`, and `wget`, but there was no evidence that the application required these tools at runtime.

Unnecessary packages increase image size, maintenance burden, and potential vulnerability surface.

### Fix

I removed the APT installation step because these tools are not required by the Node.js application.

The resulting image contains fewer unnecessary packages and has a smaller attack surface.

---

## 7. Unnecessary SSH Port — `EXPOSE 22`

### Problem

The original Dockerfile exposed port 22 even though the container runs a Node.js application and does not start an SSH server.

Exposing an unnecessary port adds confusing configuration and can encourage inappropriate SSH access to application containers.

### Fix

I removed `EXPOSE 22` and kept only `EXPOSE 3000`, which is the port used by the Node.js application.

---

## 8. Container Runs as Root

### Problem

The original Dockerfile did not specify a non-root user, so the application would run as root inside the container.

If the application were compromised, unnecessary root privileges could increase the potential impact.

### Fix

I added:

```dockerfile
USER node
```

so the application runs as the non-root `node` user provided by the official Node.js image.

This follows the principle of least privilege and limits what an attacker could do inside the container.

---

## 9. APT Package Metadata / Image Hygiene

### Problem

The original Dockerfile installed Debian packages with `apt-get` but did not clean the APT package metadata afterward.

If OS packages are installed, leaving the package lists in the image adds unnecessary data and increases image size.

### Fix

I removed the unnecessary APT installation entirely because `curl`, `vim`, and `wget` are not required by the application.

If OS packages are required in the future, the package installation and cleanup should be performed in the same Docker layer, for example:

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends <required-package> \
    && rm -rf /var/lib/apt/lists/*
```

---

# Final Dockerfile

The resulting Dockerfile is:

```dockerfile
FROM node:22-alpine@sha256:c610fcdfb1d5b4740dd70c284ed3cb16bb857e0f7166196e36a5501df7a3aa32

WORKDIR /app

COPY package*.json ./

RUN npm install -g npm@12.0.2 \
    && npm ci --omit=dev

COPY . .

USER node

EXPOSE 3000

CMD ["node", "server.js"]
```

The `npm install -g npm@12.0.2` step is retained because it was used to remediate the npm/tar vulnerability identified during Trivy scanning. `npm ci --omit=dev` separately handles installation of the application's production dependencies.


# 2. CI/CD Pipeline Audit

## 2.1 Hardcoded Credentials

### Problem

The original workflow stored the Docker Hub password and AWS secret directly in the YAML file.

Hardcoded credentials can be exposed through source control and Git history. Anyone with access to the repository or its history may be able to retrieve credentials that were previously committed.

### Fix

I removed the hardcoded credentials and replaced them with GitHub Actions Secrets.

The workflow now uses:

* `DOCKER_HUB_USERNAME`
* `DOCKER_HUB_PASSWORD`
* `SERVER_IP`
* `SSH_PRIVATE_KEY_B64`

Sensitive values are therefore provided at runtime instead of being stored in the repository.

Previously exposed credentials should also be rotated because deleting them from the current file does not remove them from Git history.

---

## 2.2 Mutable Docker Image Tag — `latest`

### Problem

The original pipeline built, pushed, and deployed the image using the `latest` tag.

A mutable tag can point to different images over time, making it difficult to determine exactly which image was deployed.

### Fix

I changed the image tag to `${{ github.sha }}`:

```yaml
docker build -t sanviag/test:${{ github.sha }} .
```

The same commit-based tag is used when pushing and deploying the image:

```yaml
docker push sanviag/test:${{ github.sha }}
```

and:

```yaml
docker pull sanviag/test:${{ github.sha }}
```

This makes each image identifiable by the Git commit that produced it and avoids relying on the mutable `latest` tag.

---

## 2.3 GitHub Actions Checkout Version

### Problem

The original workflow used:

```yaml
uses: actions/checkout@v3
```

The workflow was using an older major version of the checkout action.

### Fix

I updated it to:

```yaml
uses: actions/checkout@v6
```

For additional supply-chain protection, GitHub Actions can also be pinned to full commit SHAs and updated through a controlled dependency-update process.

---

## 2.4 Excessive GitHub Token Permissions

### Problem

The original workflow did not explicitly define the permissions available to the GitHub Actions `GITHUB_TOKEN`.

A CI/CD workflow should follow the principle of least privilege and request only the permissions it needs.

### Fix

I added:

```yaml
permissions:
  contents: read
```

The workflow only needs to read repository contents, so write permissions are not required.

---

## 2.5 Missing Container Vulnerability Scan

### Problem

The original pipeline did not scan the Docker image for known vulnerabilities before publishing and deploying it.

This could allow an image containing known CRITICAL vulnerabilities to reach production.

### Fix

I added a Trivy vulnerability scan after the Docker image is built:

```yaml
- name: Scan image with Trivy
  run: |
    trivy image \
      --scanners vuln \
      --severity CRITICAL \
      --exit-code 1 \
      --format table \
      sanviag/test:${{ github.sha }} | tee trivy-results.txt
```

The scan checks the actual Docker image built by the pipeline.

The `--severity CRITICAL` option focuses the security gate on CRITICAL vulnerabilities, while `--exit-code 1` causes the workflow to fail when matching vulnerabilities are found.

I used `--scanners vuln` because the assignment specifically requires a container vulnerability scan. This also avoids running Trivy's separate secret scanner during this step.

---

## 2.6 Trivy Scan Placement

### Why the Scan Runs After the Build

The Trivy scan is placed immediately after the Docker image is built because Trivy needs to scan the actual image that will be published and deployed.

The pipeline therefore follows this order:

```text
Checkout
   ↓
Build Docker image
   ↓
Scan image with Trivy
   ↓
Upload Trivy report
   ↓
Push image to Docker Hub
   ↓
Deploy to production
```

This is important because a failed CRITICAL vulnerability scan prevents the image from being pushed and deployed.

The scan therefore acts as a security gate between image creation and production deployment.

---

## 2.7 Trivy Report Artifact

### Problem

The original pipeline did not produce a persistent vulnerability report that could be reviewed after the workflow completed.

### Fix

I added an artifact upload step:

```yaml
- name: Upload Trivy report
  if: always()
  uses: actions/upload-artifact@v6
  with:
    name: trivy-report
    path: trivy-results.txt
```

The `if: always()` condition ensures that the report is uploaded even when the Trivy scan fails.

This allows the vulnerability findings to be reviewed from the GitHub Actions run.

---

## 2.8 Insecure SSH Host Verification

### Problem

The original deployment used:

```bash
-o StrictHostKeyChecking=no
```

This disables SSH host-key verification. If an attacker were able to intercept the connection, the workflow could potentially connect to an unexpected server without detecting a host-key mismatch.

### Fix

I removed `StrictHostKeyChecking=no`.

The workflow now obtains the server host key using:

```bash
ssh-keyscan -H "${{ secrets.SERVER_IP }}" >> ~/.ssh/known_hosts
```

and connects using:

```bash
-o StrictHostKeyChecking=yes
```

This enables SSH host-key checking and prevents the SSH client from automatically accepting an unknown host key.

---

## 2.9 SSH Private Key Protection

### Problem

The deployment requires an SSH private key. A private key should not be committed to the repository or stored directly in the workflow file.

### Fix

I stored the SSH private key as the GitHub Actions Secret:

```text
SSH_PRIVATE_KEY_B64
```

The workflow reconstructs the key at runtime:

```bash
echo "${{ secrets.SSH_PRIVATE_KEY_B64 }}" | base64 --decode > ~/.ssh/github_actions_deploy
chmod 600 ~/.ssh/github_actions_deploy
```

The key is therefore not stored in the repository.

I also added a validation step:

```bash
ssh-keygen -y -f ~/.ssh/github_actions_deploy > /dev/null
```

This verifies that the decoded file is a valid private key before attempting deployment.

---

## 2.10 Excessive Container Privileges — `--privileged`

### Problem

The original deployment used:

```bash
docker run -d --privileged -p 80:3000 myapp:latest
```

The `--privileged` option grants the container significantly more access to the host than a normal application container requires.

If the application were compromised, these additional privileges could increase the potential impact and make host-level attacks easier.

The fact that the application is only internally accessible does not remove this risk. An attacker who compromises another internal system or the application itself could still potentially exploit the excessive container privileges.

### Fix

I removed `--privileged` and added container hardening options:

```bash
--read-only
--cap-drop=ALL
--security-opt=no-new-privileges:true
```

These settings reduce the container's ability to modify its filesystem, use Linux capabilities, or gain additional privileges.

---

## 2.11 Container Lifecycle Management

### Problem

The original deployment started a new container without explicitly stopping and removing the previous container.

Repeated deployments could therefore result in old containers remaining active or cause conflicts with the container name.

### Fix

I added:

```bash
docker stop myapp 2>/dev/null || true
docker rm myapp 2>/dev/null || true
```

before starting the new container.

The container is also started with:

```bash
--name myapp
--restart unless-stopped
```

This provides predictable replacement during deployments and allows Docker to restart the application automatically after an unexpected restart.

---

## 2.12 Production Port Mapping

### Problem

The application listens on port `3000` inside the container, while the required production endpoint uses port `80`.

### Fix

The deployment maps:

```bash
-p 80:3000
```

This exposes port 80 on the EC2 host and forwards requests to port 3000 inside the container.

The Dockerfile exposes only:

```dockerfile
EXPOSE 3000
```

No SSH service is exposed by the application container.

## 2.13 Pinned Docker Base Image

### Problem

A container build can become unpredictable if its base image uses a mutable tag such as `latest`.

### Fix

The Dockerfile uses `node:22-alpine` pinned to a specific SHA256 digest. This ensures the CI build uses the exact base image that was audited and tested rather than whichever image a mutable tag points to later.

# 3. Trivy Vulnerability Scanning

## 3.1 Scan Placement

The Trivy scan is executed immediately after the Docker image is built and before the image is pushed to Docker Hub or deployed to production.

The pipeline follows this order:

```text
Checkout
   ↓
Build Docker image
   ↓
Trivy vulnerability scan
   ↓
Upload Trivy report
   ↓
Push image to Docker Hub
   ↓
Deploy to production
```

This placement is important because the scan evaluates the actual Docker image that will be published and deployed.

If the scan detects a CRITICAL vulnerability, the pipeline fails before the image can be pushed or deployed.

---

## 3.2 Fail the Pipeline on CRITICAL Vulnerabilities

The Trivy step uses:

```yaml
- name: Scan image with Trivy
  run: |
    trivy image \
      --scanners vuln \
      --severity CRITICAL \
      --exit-code 1 \
      --format table \
      sanviag/test:${{ github.sha }} | tee trivy-results.txt
```

The `--severity CRITICAL` option makes CRITICAL vulnerabilities the security gate.

The `--exit-code 1` option causes the Trivy command to return a non-zero exit code when CRITICAL vulnerabilities are detected. This causes the GitHub Actions job to fail and prevents the following push and deployment steps from running.

The `--scanners vuln` option limits this step to vulnerability scanning.

---

## 3.3 Trivy Report Artifact

The scan output is saved to:

```text
trivy-results.txt
```

The workflow uploads this file as a GitHub Actions artifact:

```yaml
- name: Upload Trivy report
  if: always()
  uses: actions/upload-artifact@v6
  with:
    name: trivy-report
    path: trivy-results.txt
```

The `if: always()` condition is important because the report should still be uploaded when the Trivy scan fails.

This allows the vulnerability findings to be reviewed after the workflow finishes.

---

## 3.4 Vulnerability Investigation — `CVE-2026-59873`

During testing, Trivy initially reported a CRITICAL vulnerability:

```text
Library:             tar
Vulnerability:       CVE-2026-59873
Severity:            CRITICAL
Installed Version:   7.5.11
Fixed Version:       7.5.19
Title:               tar: node-tar: Denial of Service via crafted gzip bomb
```

The vulnerability was initially suppressed using:

```text
.trivyignore
```

which contained:

```text
CVE-2026-59873
```

However, suppressing the vulnerability does not actually fix the vulnerable package. Therefore, I did not use the suppression as the final solution.

---

## 3.5 Remediation of the `tar` Vulnerability

I investigated where the vulnerable `tar` package was installed in the container.

The package was not an application dependency. It was included inside the npm installation:

```text
/usr/local/lib/node_modules/npm/node_modules/tar
```

The initial container contained:

```text
npm: 10.9.8
tar: 7.5.11
```

I then updated npm to a version that contains the fixed `tar` dependency.

The Dockerfile now includes:

```dockerfile
RUN npm install -g npm@12.0.2 \
    && npm ci --omit=dev
```

After rebuilding the image, I verified the npm version:

```bash
docker run --rm sanviag/test:debug npm --version
```

The result was:

```text
12.0.2
```

I also verified the version of the `tar` package inside npm:

```bash
docker run --rm sanviag/test:debug node -e "console.log(require('/usr/local/lib/node_modules/npm/node_modules/tar/package.json').version)"
```

The result was:

```text
7.5.19
```

This matches the fixed version reported by Trivy.

Therefore, the vulnerability was remediated by upgrading the dependency rather than permanently suppressing the finding.

---

## 3.6 Verification

After rebuilding the Docker image with the updated npm version, I ran Trivy again.

The final scan reported:

```text
Total: 0 (CRITICAL: 0)
```

This confirmed that the previously detected `tar` vulnerability was no longer detected in the rebuilt image.

The `.trivyignore` suppression was also removed so that the vulnerability would be detected if it reappears in a future build.

---

## 3.7 Why Suppression Was Not Used as the Final Fix

A vulnerability suppression only tells the scanner to ignore a finding. It does not remove the vulnerable software from the image.

For this assignment, the preferred approach was to remediate the vulnerability by upgrading the affected dependency and then verify the resulting image with Trivy.

Suppressions may be appropriate for documented temporary exceptions when a vulnerability cannot immediately be fixed, but they should include a clear reason, owner, and review or expiration process.

In this case, the vulnerable `tar` version could be replaced, so remediation was preferred over suppression.

# 4. Decision Questions

## Q1. Vulnerability Management

If Trivy finds three CRITICAL OpenSSL vulnerabilities in the base image, but moving to a newer base image breaks a native application module and the issue cannot be fixed within 24 hours, I would first assess whether the vulnerabilities are actually exploitable in this application and determine the exposure and business impact.

There are three possible approaches:

1. **Fix** — move to a patched base image or update the affected dependency if this can be done safely.
2. **Mitigate** — if an immediate fix is not possible, reduce exposure through controls such as network restrictions, least-privilege container settings, reduced attack surface, and limiting access to the affected functionality.
3. **Accept temporarily** — if the risk cannot be removed or sufficiently mitigated within the required timeframe, document a temporary risk acceptance with an owner, justification, and review date.

In this situation, I would not simply ignore the CRITICAL findings. I would keep the currently working base image temporarily if changing it would break production functionality, apply reasonable mitigations, document the three CVEs and their impact, notify the responsible engineer or security owner, and create a tracked remediation task with a deadline.

The decision should be based on the actual exploitability and business risk rather than only the CVSS severity.

---

## Q2. Container Security — `--privileged`

I would not consider `--privileged` safe simply because the application is only accessible internally.

The `--privileged` option gives a container significantly more access to the host system than a normal container. This can include additional Linux capabilities and access to host devices, weakening important container isolation boundaries.

If an attacker compromises the application through a vulnerability, those additional privileges could increase the potential impact of the compromise and make attacks against the Docker host more feasible.

Internal network access also does not eliminate the threat. An attacker could compromise another internal system, obtain internal credentials, or exploit the application itself.

For this application, there is no demonstrated requirement for privileged access, so I removed `--privileged` and instead used more restrictive container settings:

```bash
--read-only
--cap-drop=ALL
--security-opt=no-new-privileges:true
```

These controls follow the principle of least privilege and reduce the potential impact of a container compromise.

---

## Q3. Git History and Secrets

No. Removing the secrets from the current YAML file and rotating the credentials is necessary, but it does not mean the repository is completely clean.

Git history preserves previous versions of files. If credentials were committed in an earlier commit, someone with access to the repository may still be able to retrieve them from the commit history even after the current file has been changed.

The exposed credentials should therefore be considered compromised and rotated or revoked. The new credentials should be stored using GitHub Actions Secrets rather than committed to the repository.

I would also search the repository's Git history for the exposed credentials and determine whether the secrets were pushed to any remote repository, fork, cache, artifact, or other location.

If required by the organization's security process, the sensitive data should be removed from Git history using an appropriate history-rewriting tool. However, rewriting history does not replace credential rotation because copies of the old commits may already exist elsewhere.

The important lesson is that removing a secret from the latest commit does not undo the fact that it was previously exposed.

---

## Q4. GitHub Actions SHA Pinning

Pinning GitHub Actions to full commit SHAs provides stronger supply-chain protection because the workflow references an exact version of the action instead of a mutable tag such as `v6`.

However, manually managing SHA values can make dependency updates inconvenient.

A practical approach is to pin production workflows to commit SHAs while using an automated dependency-update process.

For example, Dependabot or another approved automation tool can monitor GitHub Actions dependencies and create pull requests when newer versions are available. The SHA can then be reviewed and updated through the normal pull-request process.

The workflow can therefore have:

```yaml
uses: actions/checkout@<commit-sha>
```

instead of:

```yaml
uses: actions/checkout@v6
```

This gives stronger protection against a mutable action tag being changed unexpectedly while still keeping updates manageable.

I would also review action updates before merging them rather than automatically accepting every update. This provides a balance between supply-chain security and maintainability.

# 5. Conclusion

The original Dockerfile and CI/CD pipeline had several security and reliability weaknesses, including mutable image versions, hardcoded credentials, unnecessary packages, root execution, excessive container privileges, insecure SSH configuration, and the absence of vulnerability scanning.

The updated implementation addresses these issues by using a pinned base image, removing hardcoded secrets, reducing the container attack surface, running the application as a non-root user, using immutable Git commit image tags, restricting GitHub Actions permissions, securing SSH host verification, removing `--privileged`, and adding Trivy as a security gate before publishing and deployment.

The Trivy investigation also demonstrated the importance of remediation rather than simply suppressing vulnerabilities. The vulnerable `tar` package was identified, its source was traced to npm, npm was upgraded, and the resulting image was verified to contain `tar` version `7.5.19`, with the final scan reporting zero CRITICAL vulnerabilities.

Security is treated as an ongoing process rather than a one-time configuration change. Future dependency, base-image, GitHub Actions, and vulnerability updates should continue to be reviewed and tested through the pipeline.

