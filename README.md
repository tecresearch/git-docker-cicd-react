markdown

# 🚀 React + Vite — Production CI/CD Pipeline with Docker & GitHub Actions

A production-ready CI/CD template for React (Vite) frontends. Automates testing, Docker image building, and deployment to any VPS on every push to `main`. No Nginx. No manual steps. Works on first push.

---

## 📋 What This Template Does

| Step | Trigger | Action |
|------|---------|--------|
| CI | Push to `main` | Install → Test → Build → Docker image → Push to Docker Hub |
| Deploy | Push to `main` | SSH into VPS → Pull image → Restart container → Health check |
| Runtime | Always | `serve` on port 80 inside Docker |

---

## 🛠️ Tech Stack

| Tool | Version | Purpose |
|------|---------|---------|
| React | 19.x | UI framework |
| Vite | 8.x | Build tool |
| Vitest | 3.x+ | Testing |
| Node.js | 22 Alpine | Runtime (Docker) |
| serve | latest | Static file server |
| Docker | 24+ | Containerization |
| Docker Hub | — | Image registry |
| GitHub Actions | — | CI/CD |
| Tailscale | — | Secure VPS access (optional) |

---

## 📁 Project Structure

your-project/
├── index.html
├── package.json
├── vite.config.js
├── Dockerfile
├── docker-compose.yml
├── .github/
│ └── workflows/
│ ├── ci.yml # Build, test, push to Docker Hub
│ └── deploy.yml # Deploy to VPS
└── src/
├── App.jsx
├── App.test.jsx
├── main.jsx
└── setupTests.js
text


---

## 🚀 How to Use This Template

### 1. Copy these files into your React + Vite project

Make sure your project has:
- ✅ `Dockerfile`
- ✅ `docker-compose.yml`
- ✅ `.github/workflows/ci.yml`
- ✅ `.github/workflows/deploy.yml`
- ✅ `src/setupTests.js`
- ✅ Tests in `src/App.test.jsx` (or any `*.test.jsx`)

### 2. Update Docker image name

In `.github/workflows/ci.yml`, change:

```yaml
IMAGE=${{ secrets.DOCKERHUB_USERNAME }}/demo-frontend

to:
yaml

IMAGE=${{ secrets.DOCKERHUB_USERNAME }}/your-app-name

In .github/workflows/deploy.yml, change:
yaml

IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/demo-frontend

to:
yaml

IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/your-app-name

3. Set up GitHub Secrets

Go to GitHub Repo → Settings → Secrets and variables → Actions and add:
Secret	Value
DOCKERHUB_USERNAME	Your Docker Hub username
DOCKERHUB_TOKEN	Docker Hub access token
VPS_HOST	Your VPS IP (or Tailscale IP)
VPS_USERNAME	SSH username (e.g., ubuntu, root)
VPS_SSH_KEY	Private SSH key (full content, no passphrase)
TAILSCALE_AUTHKEY	Tailscale auth key (optional)
4. Prepare your VPS (one-time)
bash

# SSH into your VPS
ssh your-user@your-vps-ip

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Install Docker Compose
sudo apt update && sudo apt install docker-compose-plugin -y

# Open port 80
sudo ufw allow 80/tcp
sudo ufw enable

# Add your SSH public key to ~/.ssh/authorized_keys

If using Tailscale:
bash

curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

5. Push to main
bash

git add .
git commit -m "Add CI/CD pipeline"
git push origin main

Your app will be live within 2–3 minutes.
🐳 Local Development
bash

npm install
npm run dev
npm run test
npm run build
docker compose up -d --build
curl -f http://localhost

⚙️ Workflow Details
CI Pipeline (ci.yml)

Runs on every push to main:

    Checkout code

    Set up Node.js 22

    Install dependencies (npm ci)

    Run tests (vitest)

    Build frontend (vite build)

    Login to Docker Hub

    Build Docker image with tag sha-<commit-hash>

    Push image to Docker Hub

    Also tag and push as latest

Deploy Pipeline (deploy.yml)

Runs after CI succeeds (or manually):

    (Optional) Connect to Tailscale network

    Determine image tag (sha-<sha> or latest)

    SSH into VPS

    Login to Docker Hub

    Pull the correct image

    Clone/pull repository (for docker-compose.yml)

    Stop old container

    Start new container with docker compose up -d

    Health check via curl -f http://localhost

🩺 Health Checks
yaml

healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 15s

    No /health endpoint required

    Hits root / — serve returns index.html

    Status visible via docker ps

📦 Required File Contents
Dockerfile
dockerfile

FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
RUN apk add --no-cache curl && npm install -g serve
WORKDIR /app
COPY --from=builder /app/dist ./dist
EXPOSE 80
CMD ["serve", "-s", "dist", "-l", "80"]

docker-compose.yml
yaml

services:
  frontend:
    image: ${IMAGE_NAME}:${IMAGE_TAG}
    container_name: demo-frontend
    ports:
      - "80:80"
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

.github/workflows/ci.yml
yaml

name: CI/CD

on:
  push:
    branches: [main]
    paths-ignore:
      - '**.md'
      - '.gitignore'

permissions:
  contents: read

jobs:
  build-test-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests (Vitest)
        run: npm test

      - name: Build frontend
        run: npm run build

      - name: Login to Docker Hub
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin

      - name: Build and push Docker image
        run: |
          IMAGE=${{ secrets.DOCKERHUB_USERNAME }}/demo-frontend
          TAG=sha-${{ github.sha }}
          docker build -t $IMAGE:$TAG .
          docker push $IMAGE:$TAG
          docker tag $IMAGE:$TAG $IMAGE:latest
          docker push $IMAGE:latest

.github/workflows/deploy.yml
yaml

name: Deploy to VPS

on:
  push:
    branches:
      - main
    paths-ignore:
      - '**.md'
      - '.gitignore'
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Docker image tag to deploy'
        required: false
        default: 'latest'
  workflow_run:
    workflows: ['CI/CD']
    branches: [main]
    types:
      - completed

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch' || github.event_name == 'push' || (github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'success')

    steps:
      - name: Install Tailscale
        run: curl -fsSL https://tailscale.com/install.sh | sh

      - name: Connect to Tailscale
        env:
          TAILSCALE_AUTHKEY: ${{ secrets.TAILSCALE_AUTHKEY }}
        run: |
          sudo tailscale up \
            --authkey "$TAILSCALE_AUTHKEY" \
            --accept-dns=true \
            --accept-routes=true \
            --hostname github-actions-deploy-frontend
          tailscale status

      - name: Determine image tag
        id: image_tag
        run: |
          if [ "${GITHUB_EVENT_NAME}" = "workflow_dispatch" ]; then
            echo "image_tag=${{ inputs.image_tag }}" >> "$GITHUB_OUTPUT"
          elif [ "${GITHUB_EVENT_NAME}" = "push" ]; then
            echo "image_tag=sha-${{ github.sha }}" >> "$GITHUB_OUTPUT"
          elif [ "${GITHUB_EVENT_NAME}" = "workflow_run" ]; then
            echo "image_tag=sha-${{ github.event.workflow_run.head_sha }}" >> "$GITHUB_OUTPUT"
          else
            echo "image_tag=latest" >> "$GITHUB_OUTPUT"
          fi

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1.0.0
        env:
          IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/demo-frontend
          IMAGE_TAG: ${{ steps.image_tag.outputs.image_tag }}
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
          DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          port: 22
          envs: IMAGE_NAME,IMAGE_TAG,DOCKERHUB_USERNAME,DOCKERHUB_TOKEN
          script: |
            set -e

            APP_DIR="/home/$USER/app"
            REPO_URL="https://github.com/${{ github.repository }}.git"

            if [ ! -d "$APP_DIR/.git" ]; then
              git clone "$REPO_URL" "$APP_DIR"
            else
              cd "$APP_DIR"
              git fetch origin main
              git reset --hard origin/main
            fi

            cd "$APP_DIR"
            echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
            docker pull "$IMAGE_NAME:$IMAGE_TAG"
            docker compose down --remove-orphans || true
            IMAGE_NAME="$IMAGE_NAME" IMAGE_TAG="$IMAGE_TAG" docker compose up -d

            for i in $(seq 1 30); do
              if curl -f http://localhost > /dev/null 2>&1; then
                echo "✅ Healthy on attempt $i"
                break
              fi
              if [ $i -eq 30 ]; then
                echo "❌ Unhealthy – printing logs"
                docker compose logs
                exit 1
              fi
              sleep 2
            done

src/setupTests.js
js

import '@testing-library/jest-dom';

src/App.test.jsx (example)
jsx

import React from 'react';
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import App from './App';

describe('App', () => {
  it('renders the Vite + React heading', () => {
    render(<App />);
    expect(screen.getByText(/Vite \+ React/i)).toBeInTheDocument();
  });
});

src/App.jsx (example)
jsx

function App() {
  return <h1>Vite + React</h1>;
}
export default App;

vite.config.js
js

import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/setupTests.js',
  },
})

package.json (scripts part)
json

{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest run"
  }
}

🔧 Customization

Use a different port:
yaml

ports:
  - "3000:80"

Skip Tailscale: Remove Tailscale steps from deploy.yml and use public VPS IP.

Add environment variables:
yaml

environment:
  - VITE_API_URL=https://api.example.com

❓ Common Issues
Error	Solution
Dockerfile: no such file or directory	Make sure Dockerfile is in repo root
styleText not found	Use Node.js 22 in CI and Dockerfile
toBeInTheDocument not found	Ensure setupFiles: './src/setupTests.js' in vite.config.js
React is not defined	Add import React from 'react' in test files
Port 80 already in use	Stop other services (sudo systemctl stop nginx)
VPS unreachable via SSH	Check VPS online, port 22 open, key added
✅ Verification Checklist

    GitHub Actions CI passes

    Docker image on Docker Hub

    curl -f http://your-vps-ip returns HTML

    docker ps shows healthy

📝 License

MIT — Use freely in any project.