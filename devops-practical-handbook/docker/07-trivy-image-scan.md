# Docker: Trivy Image Scanning

Source coverage: Docker slide 103.

## Scan a locally available Docker image

The slide runs Trivy as a container and mounts the host Docker socket so Trivy can inspect local images.

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest \
  image nginx:latest
```

Scan the demo image used in a Jenkins/Docker pipeline (replace with an image that exists locally or in a registry):

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest \
  image <DOCKER_IMAGE>:<TAG>
```

The command mounts `/var/run/docker.sock` into the Trivy container so it can communicate with the host Docker daemon. Treat this as a privileged operation: access to the Docker socket can grant control over the host's containers and files. Do not run untrusted scanner images with this access.

For a registry-hosted image, Trivy can scan the image reference directly without mounting the Docker socket; follow the Trivy documentation for the authentication method if the registry is private.
