# Revrag DevSecOps — Operation: Fix the Pipeline

This repository contains my solution to the Revrag DevSecOps internship assignment.

## What was changed

- Hardened the Dockerfile
- Pinned the Docker base image
- Removed hardcoded secrets
- Added non-root container execution
- Hardened the runtime container
- Replaced mutable Docker tags with Git commit SHA tags
- Secured SSH deployment
- Added Trivy vulnerability scanning
- Pipeline fails on CRITICAL vulnerabilities
- Trivy reports are uploaded as GitHub Actions artifacts

## Deliverables

- [Dockerfile](./Dockerfile)
- [Deployment Workflow](./.github/workflows/deploy.yml)
- [Security Audit](./SECURITY_AUDIT.md)
- [Reflection](./REFLECTION.md)

## CI/CD Flow

GitHub Push
    ↓
Checkout
    ↓
Build Docker Image
    ↓
Trivy Vulnerability Scan
    ↓
Upload Trivy Report
    ↓
Push Verified Image to Docker Hub
    ↓
Deploy to EC2

## Deployment

The GitHub Actions workflow builds and scans the Docker image, pushes the verified image to Docker Hub, and deploys it to the EC2 instance.

Images are tagged using the Git commit SHA rather than `latest`.
