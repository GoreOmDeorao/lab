# Source Coverage and Page-by-Page Audit Map

This file records what was reviewed and where each practical section is documented. It is a coverage map, not a claim that every theory sentence from the slide decks has been copied. Theory was intentionally excluded in line with the practical-handbook request.

## Files reviewed in this pass

| Source | Size | Review method |
|---|---:|---|
| `Docker.pptx` | 103 slides | Extracted and checked text slide by slide; practical slides mapped below. Images/diagrams were not separately transcribed. |
| `Kubernetes.pptx` | 52 slides | Extracted and checked text slide by slide; practical slides mapped below. Images/diagrams were not separately transcribed. |
| `Jenkins.pptx` | 38 slides | Extracted and checked text slide by slide; practical slides mapped below. Images/diagrams were not separately transcribed. |
| `Docker notes.txt` | 174 extracted lines | Read through, including image/container lifecycle, image tagging, network notes, and MySQL no-volume example. |
| `Assignment_2.pdf` | 3 pages | Rendered all pages and checked extracted text. |
| `Assignment_3.pdf` | 2 pages | Rendered all pages and checked extracted text. |
| `Assignment_4.pdf` | 3 pages | Rendered all pages and checked extracted text. |
| `Assignment_5.pdf` | 4 pages | Rendered all pages and checked extracted text. |
| `Assignment_6.pdf` | 4 pages | Rendered all pages and checked extracted text. |

The five assignment PDFs total 16 pages. The seven PowerPoint decks total 355 slides, but this pass only re-audited Docker, Kubernetes, and Jenkins (193 slides); the previously retained Terraform, Prometheus, Chef/Puppet/Ansible, and AWS decks were not re-audited in this pass.

## Docker slide map (103 slides)

| Slides | Subject | Handbook destination / handling |
|---|---|---|
| 1-8 | Introductory Docker/container concepts, standards, and comparisons | Theory/overview excluded. |
| 9-13 | Docker Ubuntu install, operations view, Dev workflow, GitHub clone/build/run | `docker/01-container-basics.md` |
| 14-15 | Docker Engine architecture and containerd roles | Architecture theory excluded. |
| 16-25 | Start container, common images, run modes, lifecycle, image commands, prune, inspect, naming, logs | `docker/01-container-basics.md` |
| 26-38 | Container IPs, port publishing, bridge networks, BusyBox name/IP tests, custom network, Python-to-MySQL connection | `docker/02-networking.md` |
| 39-45 | Container ephemeral data, volumes, volume syntax, bind mount comparison, volume commands/lifecycle | `docker/03-volumes-and-persistence.md` |
| 46-53 | MySQL without volume vs with named volume, SQL table/data verification, cleanup | `docker/03-volumes-and-persistence.md` |
| 54-57 | NGINX website bind mount, live host file changes, inspect file in container | `docker/03-volumes-and-persistence.md` |
| 58-75 | Dockerfile instructions, Python/Java/C applications, image push/save/load, interactive input, Tomcat/httpd examples | `docker/04-dockerfiles-and-apps.md` |
| 76-77 | Tomcat and MySQL on a custom network with persistent host directories | `docker/02-networking.md` |
| 78-95 | Compose concepts and commands, Flask/MySQL app, Compose YAML construction, Java Compose example | `docker/05-compose.md` |
| 85 | NGINX + phpMyAdmin Compose repo referenced but YAML not included in the slide | Identified as an external repo; not recreated because source YAML was not included. |
| 96-101 | Swarm initialization, services, replicas, image update, registry, scale | `docker/06-swarm.md` |
| 102 | One-shot Alpine command | `docker/01-container-basics.md` |
| 103 | Trivy image scan | `docker/07-trivy-image-scan.md` |

## Kubernetes slide map (52 slides)

| Slides | Subject | Handbook destination / handling |
|---|---|---|
| 1-17 | Kubernetes rationale, architecture, control plane/worker responsibilities, conceptual workflow | Theory excluded. |
| 18 | Minikube Ubuntu installation and Docker driver | `kubernetes/01-minikube-setup.md` |
| 19 | Direct Pod versus Pod via Deployment | Practical implication covered in `kubernetes/02-pods-deployments.md`; comparison theory omitted. |
| 20-22 | Create/inspect Pod, Deployment, logs/exec/YAML, scale, self-healing deletion | `kubernetes/02-pods-deployments.md` |
| 23 | YAML section divider | No independent practical content. |
| 24-30 | Imperative-to-YAML workflow, generated YAML, Pod/Deployment manifests, labels/selectors, scale | `kubernetes/03-yaml-manifests-and-services.md` |
| 31-37 | Service types and request path | Practical service commands in `kubernetes/03-yaml-manifests-and-services.md`; diagrams/theory omitted. |
| 38-43 | NodePort manifest, selector failure, ClusterIP test, combined manifest, external static-site repo access | `kubernetes/03-yaml-manifests-and-services.md`; demo repo instructions retained with missing repo internals flagged. |
| 44-46 | Rolling-update behavior and maxSurge/maxUnavailable | `kubernetes/04-rollouts-and-troubleshooting.md` |
| 47-49 | Apply rolling update, simulate bad image, rollback, external demo repository | `kubernetes/04-rollouts-and-troubleshooting.md` |
| 50 | Wrap-up divider | No independent practical content. |
| 51-52 | Troubleshooting checks and kubectl cheat sheet | `kubernetes/04-rollouts-and-troubleshooting.md` |

## Jenkins slide map (38 slides)

| Slides | Subject | Handbook destination / handling |
|---|---|---|
| 1-3 | Jenkins overview and CI/CD flow | Theory/overview excluded. |
| 4 | Jenkins installation and unlock flow | `jenkins/01-installation-and-freestyle.md`; current Java 21+ and APT commands are checked against the official Jenkins page. |
| 5 | Freestyle versus Pipeline comparison | Theory excluded; both practical job types are covered in their respective guides. |
| 6-11 | Freestyle job, shell variables, Python/Java, script files, output redirection | `jenkins/01-installation-and-freestyle.md` |
| 12-13 | Pipeline job and workspace file assembly/append | `jenkins/02-pipelines-github-and-parameters.md` |
| 14-18 | GitHub/Docker plugins, repository clone/push, Jenkins SCM job, Poll SCM cron | `jenkins/02-pipelines-github-and-parameters.md` |
| 19-22 | Python largest-number program, Jenkins parameters, Pipeline from SCM | `jenkins/02-pipelines-github-and-parameters.md` |
| 23-31 | Jenkins Docker permissions, Docker Pipeline plugin, repo build, Docker Hub credentials, build/push | `jenkins/03-docker-ci-cd.md` |
| 31 | Source suggests `chmod 777 /var/run/docker.sock` | Explicitly warned against; not presented as a command to execute. |
| 32-38 | NGINX web app CI/CD, Dockerfiles, HTML folder, checkout/build/push/deploy Jenkinsfile | `jenkins/03-docker-ci-cd.md` |

## Assignment map

| Assignment | Pages | Questions | Destination |
|---|---:|---|---|
| Assignment 2 | 3 | Q1-Q7: image/container lifecycle, tags, push, port mapping, networks | `assignments/assignment-2-docker-basics.md` |
| Assignment 3 | 2 | Q1-Q8: volume persistence, Python and Java Dockerfiles, MySQL volume, NGINX Dockerfile, bind mount, shell script, shared volume | `assignments/assignment-3-docker-volumes-and-dockerfiles.md` |
| Assignment 4 | 3 | Q1-Q5: Compose NGINX, Python, MySQL, three NGINX containers, Minikube | `assignments/assignment-4-compose-and-minikube.md` |
| Assignment 5 | 4 | Q1-Q5 practical Swarm; Q6-Q7 are written theory prompts | `assignments/assignment-5-docker-swarm.md` (Q6-Q7 intentionally left out of this practical-only handbook) |
| Assignment 6 | 4 | Q1-Q9: Apache Pod/Deployment/scaling, YAML, services, Flask, rolling update | `assignments/assignment-6-kubernetes.md` |

## Corrections, edge cases, and source gaps

1. Docker networking lecture code used `host="localhost"` from inside a Python container while MySQL ran in a different container. The usable corrected example connects to the MySQL container name on a user-defined bridge network.
2. A few slide snippets contained smart quotes, inconsistent names, or mismatched example cleanup names. These were normalized to valid shell/Python/SQL/Groovy syntax where the intended operation was clear.
3. The Docker Compose exercise that asks for one volume mounted at the same NGINX document-root path in three containers while showing different pages is not achieved by a naive shared directory: all three would see the same files. The Compose lab uses one shared named volume with three subdirectories and three NGINX `root` settings, preserving the same volume mount target while making each page distinct.
4. The source Jenkins slides use inconsistent Docker Hub credential IDs and suggest `chmod 777 /var/run/docker.sock`. The guide uses one consistent Jenkins credential ID and explicitly warns against world-writable Docker sockets.
5. The Minikube LoadBalancer exercise needs `minikube tunnel` running; without it, an external IP can stay `pending`.
6. The Flask Kubernetes assignment builds versioned images inside Minikube so Pods can use the local image without depending on a registry. Version tags `v1` and `v2` make the rolling update explicit.
7. Source slides refer to external repositories and files without including their full contents (e.g. Docker slide 85, Kubernetes slides 42/49). They are identified as external references rather than represented as source-complete labs.
8. Images and diagrams in the three PPTX decks were not transcribed in this pass, as permitted by the request. The coverage above is based on the extracted slide text and the supplied assignment PDF text/pages.
9. The non-Docker/Kubernetes/Jenkins guides (Terraform, AWS, Prometheus, Chef/Puppet/Ansible, Apptainer) are carried forward from the previous ZIP and have not been page-by-page re-audited in this pass.
