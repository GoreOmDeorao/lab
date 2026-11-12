# DevOps Practical Handbook

A Linux-friendly, GitHub-ready practical handbook assembled from the supplied CDAC material. It focuses on **commands, configuration files, scripts, lab execution, verification, and cleanup**. Long theory-only sections are omitted.

## Start here

1. Read this README once.
2. Open the assignment file for the lab you need.
3. Follow the commands in order and create files with the exact filenames shown.
4. Replace every placeholder such as `YOUR_PRN`, `YOUR_NAME`, `your-dockerhub-username`, and repository URLs before running commands.
5. Read any **DESTRUCTIVE** warning before executing cleanup commands.

## Repository contents

### Assignment solutions

| Assignment | Practical solution |
|---|---|
| Assignment 2 | [Docker image/container basics, tags, Docker Hub, ports, networks](assignments/assignment-2-docker-basics.md) |
| Assignment 3 | [Docker volumes, Python/Java/C/shell images, NGINX, bind mounts](assignments/assignment-3-docker-volumes-and-dockerfiles.md) |
| Assignment 4 | [Compose NGINX/Python/MySQL/three NGINX containers and Minikube setup](assignments/assignment-4-compose-and-minikube.md) |
| Assignment 5 | [Docker Swarm services, worker, scaling, rolling update, cleanup](assignments/assignment-5-docker-swarm.md) |
| Assignment 6 | [Kubernetes Pods, Deployments, Services, Flask Deployment, rolling update](assignments/assignment-6-kubernetes.md) |

### Docker

- [Installation, images, containers, lifecycle, tags, Docker Hub](docker/01-container-basics.md)
- [Networks, DNS, Python/MySQL and Tomcat/MySQL communication](docker/02-networking.md)
- [Named volumes, bind mounts, MySQL persistence, shared storage](docker/03-volumes-and-persistence.md)
- [Dockerfiles and Python, Java, C, shell, NGINX, Apache, Tomcat examples](docker/04-dockerfiles-and-apps.md)
- [Compose static sites, Python, MySQL, Java, shared-volume NGINX](docker/05-compose.md)
- [Swarm services, scaling, rolling updates, rollback](docker/06-swarm.md)
- [Trivy image scanning](docker/07-trivy-image-scan.md)

### Kubernetes

- [Ubuntu Minikube setup and cluster commands](kubernetes/01-minikube-setup.md)
- [Pods, Deployments, scaling, self-healing](kubernetes/02-pods-deployments.md)
- [YAML manifests, ClusterIP, NodePort, LoadBalancer](kubernetes/03-yaml-manifests-and-services.md)
- [Rolling updates, rollback, troubleshooting, kubectl reference](kubernetes/04-rollouts-and-troubleshooting.md)

### Jenkins

- [Ubuntu installation and Freestyle jobs](jenkins/01-installation-and-freestyle.md)
- [Pipelines, GitHub checkout, Poll SCM, parameters](jenkins/02-pipelines-github-and-parameters.md)
- [Docker access, Jenkins credentials, image build/push/deploy](jenkins/03-docker-ci-cd.md)

### Other earlier topics retained from the previous reference ZIP

These guides were carried forward, but were **not re-audited page by page in this pass**:

- [Terraform](terraform/terraform-labs.md)
- [AWS EC2](aws/01-ec2.md)
- [AWS S3, Lambda, DynamoDB](aws/02-s3-lambda.md)
- [AWS VPC](aws/03-vpc.md)
- [Prometheus and Grafana](prometheus/prometheus-grafana.md)
- [Chef](config-management/chef.md)
- [Puppet](config-management/puppet.md)
- [Ansible](config-management/ansible.md)
- [Apptainer](apptainer/apptainer.md)

## How the examples are written

- Each lab states which folder/files to create before showing their contents.
- Shell commands are in `bash` blocks; application source and configuration are in language-specific blocks.
- Explanatory notes focus on what a command changes and how to verify its result.
- Examples include a verification step and, where appropriate, a cleanup step.
- Commands that delete containers, volumes, clusters, or all matching resources are marked **DESTRUCTIVE**.
- Replace placeholder usernames, PRNs, repository names, passwords, and IP addresses. Do not commit actual passwords, Docker Hub tokens, cloud keys, SSH private keys, or local `.env` files.

## Clone and use on Linux

```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/devops-practical-handbook.git
cd devops-practical-handbook
cat README.md
find . -maxdepth 2 -type f | sort
```

If you downloaded the ZIP instead of cloning:

```bash
unzip devops-practical-handbook.zip
cd devops-practical-handbook
cat README.md
```

## Commit this folder to GitHub

From inside the extracted repository folder:

```bash
git init
git branch -M main
git add .
git status                         # Review files before committing
git commit -m "Add DevOps practical handbook"
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/<REPOSITORY_NAME>.git
git push -u origin main
```

## Important caveats

- These are lab instructions, not production deployment templates. Example passwords such as `root123` are for isolated local labs only.
- Docker-group access and Docker socket access are highly privileged. The Jenkins guide intentionally does not recommend `chmod 777 /var/run/docker.sock`.
- Minikube needs Docker running and enough host resources. A LoadBalancer Service on Minikube needs `minikube tunnel` to remain running.
- A locally built Docker image must be available to the Kubernetes node. The Flask assignment builds the image into Minikube and uses a versioned image tag.
- Several source slides reference external GitHub repositories but do not include their full file contents. Those missing external files are identified rather than fabricated.
- Container image tags such as old Tomcat versions can become unavailable or unsupported. Where the lecture used an old tag, it is identified as a legacy example.

## Source coverage and audit notes

See [SOURCE_COVERAGE.md](SOURCE_COVERAGE.md) for the slide/page-to-file mapping, the scope of what was scanned, and source gaps.

## Official documentation links

- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Jenkins on Linux: https://www.jenkins.io/doc/book/installing/linux/
- Minikube installation: https://minikube.sigs.k8s.io/docs/start/
- Minikube Docker driver: https://minikube.sigs.k8s.io/docs/drivers/docker/
- Kubernetes kubectl: https://kubernetes.io/docs/reference/kubectl/
