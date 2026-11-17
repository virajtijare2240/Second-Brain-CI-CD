# Jenkins CI/CD Implementation Guide — Second Brain

## Pipeline scope

This pipeline automates the existing Second Brain application. It checks out the Git repository, installs backend dependencies, runs the existing Pytest suite, builds the React/Vite frontend, builds backend and frontend Docker images, optionally pushes images to Docker Hub, and deploys the containers locally with Docker Compose.

## Files added

- `Jenkinsfile` — declarative Jenkins pipeline for a Windows Jenkins agent.
- `backend/Dockerfile` — packages the FastAPI backend.
- `frontend/Dockerfile` and `frontend/nginx.conf` — builds the React frontend and serves it with Nginx; `/api` and `/health` are proxied to the backend container.
- `docker-compose.yml` — runs frontend and backend together with persistent SQLite and upload volumes.
- `.dockerignore` — excludes local dependencies, databases, uploads, and secrets from Docker build contexts.

## Prerequisites on the Jenkins agent

Install and verify:

1. Git for Windows.
2. Python 3.10 or newer, available as `py` in Command Prompt.
3. Node.js 18 or newer, available as `npm.cmd`.
4. Docker Desktop with the Docker Compose v2 plugin. Docker Desktop must be running.
5. Jenkins with Pipeline, Git, and JUnit plugins.

The Jenkins service account must be able to execute `git`, `py`, `npm.cmd`, `docker`, and `docker compose`. If Jenkins was installed as a Windows service, restart the service after changing its PATH.

## Add project to GitHub

Commit the new files and push them to the same repository as the application. Do not commit `.env`, API keys, SQLite databases, virtual environments, `node_modules`, or uploaded user documents. The repository `.gitignore` excludes these local files.

## Create Jenkins job

1. Open the Jenkins dashboard.
2. Select **New Item**.
3. Enter `Second-Brain-CI-CD`.
4. Select **Pipeline** and click **OK**.
5. Under Pipeline, select **Pipeline script from SCM**.
6. Select **Git**, enter the GitHub repository URL, and configure credentials if the repository is private.
7. Set branch to `*/main` (or the actual default branch).
8. Set Script Path to `Jenkinsfile` and save.
9. Select **Build Now** for the first run.

For automatic builds after pushes, configure a GitHub webhook pointing to `http(s)://<JENKINS_HOST>/github-webhook/` and enable the GitHub hook trigger in the job. Jenkins must be reachable by GitHub; localhost-only Jenkins needs a secure tunnel or another reachable host. Do not expose an unsecured Jenkins instance to the public internet.

## Docker Hub (optional)

1. Create two repositories in Docker Hub named `second-brain-backend` and `second-brain-frontend`.
2. In Jenkins, open **Manage Jenkins → Credentials → (global) → Add Credentials**.
3. Choose **Username with password**. Use your Docker Hub username and an access token as the password.
4. Set the credential ID to exactly `dockerhub-creds`.
5. Run the pipeline with **PUSH_TO_DOCKERHUB** checked.

Leave this parameter unchecked for a local-only build and deployment. Never put the Docker Hub token in the Jenkinsfile or source repository.

## Verify deployment

After a successful pipeline, open:

- Frontend: `http://localhost:8080`
- Backend health: `http://localhost:8000/health`
- Swagger API docs: `http://localhost:8000/docs`

Useful commands in a terminal:

```bat
docker compose ps
docker compose logs --tail=100 backend
docker compose logs --tail=100 frontend
```

Stop the deployment with `docker compose down`. Persistent database and uploaded files are held in named Docker volumes and are not removed by this command. To delete the persistent data as well, `docker compose down -v` will remove those volumes; use it only when you intentionally want to erase the deployed local data.

## Pipeline stages

1. **Checkout** — gets the latest source from GitHub.
2. **Backend Setup and Tests** — creates an isolated Python environment, installs `backend/requirements.txt`, and runs `pytest` with a JUnit report.
3. **Frontend Build** — runs `npm ci` and `npm run build`.
4. **Build Docker Images** — builds versioned and `latest` images for both services.
5. **Push Images to Docker Hub** — optional; runs only when the build parameter is enabled and Jenkins credential `dockerhub-creds` exists.
6. **Deploy Locally** — starts the application with Docker Compose.

## Important notes

- The provided Jenkinsfile uses Windows `bat` steps. Use a Windows Jenkins agent with the prerequisites above. A Linux agent requires equivalent `sh` steps and Linux tool paths.
- The ZIP contained local `.env` and SQLite files. They were deliberately excluded from this CI/CD copy so secrets and local data are not committed or baked into images. Supply production secrets through Jenkins credentials or deployment environment variables.
- The Compose file has a development fallback `SECRET_KEY` so the demo can start. Replace it with a strong secret before any shared or public deployment, for example by setting `SECRET_KEY` in the environment used to run Docker Compose.
- The backend's existing test fixture sets a test SQLite URL. The pipeline does not need a live production database to run tests.
- The current pipeline deploys locally on the Jenkins machine. A remote production deployment needs a target host, credentials, and a deployment strategy agreed with the project owner.
