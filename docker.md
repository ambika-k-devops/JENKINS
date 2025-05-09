# Top 20 Docker Scenario-Based Interview Questions (Enhanced for 4+ Years Experience)

## 1. Container keeps restarting – how do you fix it?
In one of our dev environments, a Node.js container was stuck in a crash loop. Using `docker ps -a`, I found the container kept exiting with code 1. I ran `docker logs <container_id>` and identified the error — it was due to a missing environment variable. I fixed the variable in the Docker Compose file, restarted the service, and the container stabilized. This type of issue is common when configuration isn't validated before deployment. We later added a pre-deployment config check script to catch such issues earlier.

## 2. Docker container not able to access the internet – what will you check?
We faced this in staging where a container couldn’t reach external APIs. I entered the container using `docker exec` and tried `ping 8.8.8.8` and `curl google.com`. It failed, confirming DNS issues. The container was on the default bridge network. I fixed it by adding a custom bridge network and configured Docker daemon to use Google's DNS via `/etc/docker/daemon.json`. Restarted the Docker service, and everything worked fine. Later, we used `network_mode: host` in critical network-bound services to avoid isolation.

## 3. Docker image too large – how will you optimize?
Our Spring Boot image was 1.2GB. This was slowing builds and pushing large layers to the registry. I rewrote the Dockerfile using a multi-stage build. The first stage compiled the JAR, and the second used a minimal `openjdk-alpine` image. We also removed build tools and cleaned up cache files. This reduced our image size to 250MB. The deployment time reduced significantly, and ECR storage bills went down too.

## 4. Code updated, but container still runs old code – why?
A Python app container was serving stale code. I realized Docker was caching previous layers. I used `--no-cache` during build and moved the `COPY` instruction below the `RUN pip install` step. This ensured dependencies were cached but new source code was always copied fresh. Post this fix, our CI pipeline was updated to force clean builds during major releases.

## 5. Two containers can’t talk – how do you fix it?
In a Docker Compose setup, the app couldn’t reach MySQL. I checked that both containers were in the same Docker-defined network. The app was trying to connect using `localhost`. I updated the DB host to the service name (`db`) and restarted the stack. Docker's internal DNS resolved the name. We later introduced a shared overlay network in Swarm for multi-host communication.

## 6. Data gets lost after container restart – what to do?
A MySQL container would lose all data when restarted. I realized the data directory wasn’t persisted. I added a named volume like `mysql_data:/var/lib/mysql` in Docker Compose. After that, even after recreating the container, the data remained intact. We also scheduled weekly volume backups to S3 for disaster recovery.

## 7. Dockerfile changes not reflecting – what's the fix?
We updated `package.json`, but `npm install` wasn't picking new packages. Docker reused the cached layer. I changed the order in the Dockerfile to `COPY package.json` → `RUN npm install` → `COPY . .`. This leveraged caching for dependencies and rebuilt app code properly. We also enabled `--no-cache` in nightly builds to avoid stale layers.

## 8. App is running but not accessible via browser – what’s the issue?
The app inside the container was listening on `localhost`, which doesn’t expose to host machine. I updated it to bind `0.0.0.0`, and ran the container with `-p 8080:8080`. Also verified host firewall rules. After that, the app was accessible externally. We documented this as a checklist item for all new Dockerized apps.

## 9. Docker Compose service failing to start – how did you fix it?
The app container failed to connect to DB with 'connection refused'. Logs showed it tried connecting before DB was ready. I added `depends_on` and a `wait-for-it.sh` script to delay the app startup until the DB port was open. This stabilized the startup sequence. We also added health checks for DB to make it more robust.

## 10. Builds are slow due to Docker caching – how did you improve?
Each build was reinstalling dependencies, wasting time. I restructured the Dockerfile: first copied `package.json`, then ran `npm install`, then copied source code. As dependencies changed less frequently, Docker reused that layer. This reduced our build time from 4 mins to 1.5 mins in CI/CD.

## 11. How do you update a container in production without downtime?
We used a rolling deployment strategy. Behind a load balancer, I deployed a new container, verified health checks, and only then removed the old version. No downtime was observed. In Kubernetes, this was handled using `RollingUpdate` strategy in Deployment YAML.

## 12. Container exits immediately – what steps do you take?
One backend container kept exiting. I used `docker logs` and found a missing `config.json`. The volume was incorrectly mounted. Fixed the path in Compose file, rebuilt the container, and confirmed the issue was resolved. To prevent recurrence, we added a pre-run validation step to check mounted file presence.

## 13. How do you troubleshoot inside a running container?
I used `docker exec -it <container> /bin/sh` to explore the container. Used `ls`, `cat`, `ps`, `top` to check logs, running processes, and file contents. For Python apps, I checked `gunicorn` logs and confirmed dependencies using `pip list` inside the container.

## 14. No logs showing for a running container – what could be wrong?
The app was configured to write logs to a file instead of STDOUT. I updated the logging config to write to `stdout` so Docker could capture it. This enabled `docker logs` and integrated well with centralized logging using ELK.

## 15. How do you share data between two containers?
We needed our app to write logs and a separate container to parse them. I created a named volume and mounted it to both containers. This allowed real-time access to logs. This method was later used for audit logs and file processing as well.

## 16. How do you manage secrets in Docker?
In dev, we used `.env` files. But in production, we used Docker Swarm secrets, which mounted sensitive values as in-memory files with strict permissions. It reduced the risk of leaking credentials inside image layers or logs.

## 17. How do you clean unused Docker images, volumes, and containers?
On our GitLab runners, disk usage was high due to old layers. I scheduled `docker system prune -f`, `docker volume prune -f`, and `docker image prune -a -f`. Also added registry cleanup rules to remove images older than 15 days. Alerts were integrated for disk usage using Prometheus + Alertmanager.

## 18. How do you identify which image or layer is consuming most space?
I used `docker image ls`, `docker image inspect`, and `docker history`. This showed which layers were largest — usually caused by large packages or `COPY .` including unnecessary files. We then optimized by creating `.dockerignore` and using minimal base images.

## 19. How do you test Docker images before pushing to production?
I tagged the image locally as `myapp:dev`, ran it using `-p` to test endpoints with Postman. Also checked logs, volume mounts, and env vars. After testing, I tagged it as `latest` and pushed it to the container registry.

## 20. Difference between build-time and run-time variables – how do you handle them?
Build-time variables are passed using `--build-arg` and are only used during image creation. Runtime variables are passed via `-e` or Compose files, used during container execution. We clearly separated them in Dockerfile using `ARG` and `ENV`.
