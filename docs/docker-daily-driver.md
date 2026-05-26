# docker daily-driver

The commands you'll actually reach for on any working day with Docker — building, running, inspecting,
logging, cleaning up. CLI only (no `docker compose`).

## 1. Build images

Build a tagged image from a Dockerfile in the current directory:

```bash
docker build -t nginx:alpine -f Dockerfile .
```

Build without using any cached layers (forces every step to re-run):

```bash
docker build --no-cache -t nginx:alpine -f Dockerfile .
```

Pass build-time variables and pull the latest base image:

```bash
docker build --pull --build-arg NODE_ENV=production -t nginx:alpine .
```

## 2. Run containers

Run a detached, auto-removed container with a name, port and volume mount, plus an env var:

```bash
docker run -d --rm --name my-app \
  -p 8080:80 \
  -v ./data:/data \
  -e LOG_LEVEL=info \
  nginx:alpine
```

Attach to an existing user-defined network:

```bash
docker run -d --name my-app --network app-net nginx:alpine
```

## 3. Run an interactive shell

One-shot interactive container, auto-removed on exit — handy for poking at an image:

```bash
docker run -it --rm nginx:alpine /bin/sh
```

Try `/bin/bash` first on Debian/Ubuntu-derived images; fall back to `/bin/sh` for Alpine / distroless.

## 4. Exec into a running container

Attach an interactive shell to a container that's already running:

```bash
docker exec -it my-app /bin/sh
```

Run a one-off command without a TTY (good for scripts and CI):

```bash
docker exec my-app cat /etc/os-release
```

## 5. Inspect

List running containers (add `-a` for stopped ones too):

```bash
docker ps
docker ps -a
```

Get the full low-level JSON for a container or image:

```bash
docker inspect my-app
```

Show running processes inside a container, plus a live resource usage stream:

```bash
docker top my-app
docker stats my-app
```

## 6. Logs

Tail the last 100 lines and follow new output:

```bash
docker logs --tail 100 -f my-app
```

Logs from the last 15 minutes, with timestamps:

```bash
docker logs --since 15m --timestamps my-app
```

## 7. Stop and remove

Graceful stop (sends SIGTERM, waits, then SIGKILL):

```bash
docker stop my-app
```

Remove a stopped container, including its anonymous volumes:

```bash
docker rm -v my-app
```

Force-remove a running container in one step:

```bash
docker rm -f my-app
```

Send a specific signal (e.g. SIGHUP to make NGINX reload config):

```bash
docker kill -s HUP my-app
```

## 8. Image management

List images and pull/push by reference:

```bash
docker images
docker pull nginx:alpine
docker push nginx:alpine
```

Re-tag an image for a different registry, then push:

```bash
docker tag nginx:alpine registry.example.com/team/nginx:alpine
docker push registry.example.com/team/nginx:alpine
```

Remove an image (use `-f` if something still references it):

```bash
docker rmi nginx:alpine
```

## 9. Volumes and networks

Named volumes survive container removal — use them for data you want to keep:

```bash
docker volume ls
docker volume create my-app-data
docker volume rm my-app-data
```

User-defined bridge networks give containers DNS-based discovery by name:

```bash
docker network ls
docker network create app-net
docker network rm app-net
```

## 10. Cleanup

Remove stopped containers only:

```bash
docker container prune -f
```

Remove unused images (add `-a` to include any image not currently referenced by a container):

```bash
docker image prune -a -f
```

Nuke everything not currently in use — containers, networks, images, build cache, and anonymous volumes:

```bash
docker system prune -a --volumes -f
```

This is destructive. Without `-a` only dangling images are removed; without `--volumes` named volumes
that nothing references are kept.
