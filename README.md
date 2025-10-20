Nodejs-web application
This repository contains helper scripts and a CI/CD workflow for building, publishing, and deploying a Node.js app to a remote server using Docker.

Contents
- `MAIN.YAML` — GitHub Actions workflow that builds a Docker image, pushes it to Docker Hub, and deploys to a remote server via SSH.

> Note: The instructions below assume a Debian/Ubuntu environment for installers and an Ubuntu GitHub Actions runner (as in `MAIN.YAML`). Adjust paths and commands for other OSes.

## Quick install (tools)

If you need Google Cloud SDK + GKE plugin + kubectl (for GKE deployments), use the included script or follow the manual steps.

## CI/CD workflow (from `MAIN.YAML`)

The repository contains a GitHub Actions workflow (`MAIN.YAML`) that runs on pushes to the `main` branch. It performs the following steps in the `build-test-deploy` job:

1. Checkout code
2. Setup Node.js 18
3. Install dependencies (`npm install`)
4. Run tests (`npm test`)
5. Build a Docker image and tag it as `${{ secrets.DOCKER_USERNAME }}/my-node-app:latest`
6. Log into Docker Hub using `${{ secrets.DOCKER_USERNAME }}` and `${{ secrets.DOCKER_PASSWORD }}`
7. Push the Docker image to Docker Hub
8. Deploy to a remote server via SSH using `appleboy/ssh-action`:
   - Pull the newly pushed image
   - Stop and remove any existing `my-node-app` container
   - Run the container in detached mode mapping host port 3000 -> container port 3000

### GitHub Secrets required
Set the following repository secrets in GitHub Settings → Secrets & variables → Actions:

- `DOCKER_USERNAME` — Docker Hub username
- `DOCKER_PASSWORD` — Docker Hub password (or access token)
- `HOST` — IP or hostname of the remote server
- `USERNAME` — SSH username for the remote server
- `SSH_PRIVATE_KEY` — Private key for SSH access (the action uses this to authenticate)

### How deployment works (server-side)

On the remote server the workflow runs these commands via SSH (see `MAIN.YAML`):

```bash
docker pull ${DOCKER_USERNAME}/my-node-app:latest
docker stop my-node-app || true
docker rm my-node-app || true
docker run -d -p 3000:3000 --name my-node-app ${DOCKER_USERNAME}/my-node-app:latest
```

Make sure the remote server has:

- Docker installed and running
- The SSH user has permission to run Docker (either via sudoers or by being in the `docker` group)
- Open port 3000 or an appropriate firewall rule allowing inbound traffic

## Recommended server setup (quick)

On the remote Ubuntu server, run:

```bash
# Add your SSH key locally (if not already)
# On your machine:
ssh-copy-id -i ~/.ssh/id_rsa.pub user@server

# On the server (or via provisioning):
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER  # logout/login or restart required
sudo systemctl enable --now docker
```

If you don't want to give your SSH user Docker permissions, you can modify the GitHub Action to run docker commands with `sudo` (and configure sudoers accordingly).

## Local testing of the Docker image

Build locally and run:

```bash
docker build -t my-node-app:local .
docker run --rm -p 3000:3000 my-node-app:local
```

## Troubleshooting

- Docker login failures: check that `DOCKER_USERNAME` and `DOCKER_PASSWORD` are correct. Consider using a Docker Hub access token.
- SSH failures: ensure `HOST`, `USERNAME`, and `SSH_PRIVATE_KEY` are correct and the SSH key is allowed on the server.
- Port in use: if port 3000 is already in use on the server, change the host port mapping in the workflow or stop the conflicting service.

---
