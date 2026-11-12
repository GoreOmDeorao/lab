# DevOps Practical Reference

Practical-only notes extracted from the supplied CDAC slide decks and `Docker notes.txt`. Theory slides have been left out. Each technology is kept in its own file so you can browse this repository on a Linux machine and copy the commands, configuration, and scripts for a lab.

## Contents

### Docker
- [01 — CLI and container lifecycle](docker/01-cli-and-lifecycle.md)
- [02 — Networking](docker/02-networking.md)
- [03 — Volumes and persistent data](docker/03-volumes.md)
- [04 — Dockerfiles, images, and registry](docker/04-images-and-dockerfiles.md)
- [05 — Compose, Swarm, and Trivy](docker/05-compose-swarm-security.md)

### Kubernetes
- [Minikube, Pods, Deployments, Services, YAML, rollout, and rollback](kubernetes/kubernetes-labs.md)

### Terraform
- [Terraform practical labs](terraform/terraform-labs.md)

### Jenkins
- [Jenkins jobs, GitHub, pipelines, and Docker](jenkins/jenkins-labs.md)

### Prometheus and Grafana
- [Prometheus app monitoring and Grafana dashboard](prometheus/prometheus-grafana.md)

### Configuration management
- [Chef recipes](config-management/chef.md)
- [Puppet manifests](config-management/puppet.md)
- [Ansible commands and inventories](config-management/ansible.md)

### AWS
- [EC2, user data, AMIs, and SSH](aws/01-ec2.md)
- [S3, Lambda, and DynamoDB lab steps](aws/02-s3-lambda.md)
- [VPC public/private subnet lab](aws/03-vpc.md)

### Apptainer
- [Install, run, and build a `.sif` image](apptainer/apptainer.md)

A root `.gitignore` is included to help keep Terraform state, local virtual environments, private keys, and build artifacts out of Git commits. Review it before pushing your lab repository.

## How to use these files

1. Open the relevant Markdown file on your Linux machine.
2. Replace every placeholder such as `<YOUR_GITHUB_USERNAME>`, `<PRN>`, `<BUCKET_NAME>`, `<AMI_ID>`, and `<PRIVATE_IP>` before running commands.
3. Run commands in the order shown. Code blocks labeled `bash`, `yaml`, `hcl`, `python`, `ruby`, `groovy`, `json`, or `sql` contain the corresponding file contents or shell commands.
4. Commands marked **Cleanup** or **Destructive** delete resources or data. Read them before running.

## Important accuracy notes

- The examples are based on the supplied slides and Docker notes. Slide/page references are included near each lab so you can trace an item back to the material.
- Obvious transcription problems (smart quotes, malformed shell options, mismatched local ports, and inconsistent example container names) have been corrected where the intended command is unambiguous.
- Incomplete examples are labeled as fragments. Where the slides refer to a YAML file or external code but do not provide its contents, this repository does not invent that missing source code.
- AWS resources can incur charges. Delete lab resources after use. Never commit AWS access keys, private SSH keys, passwords, Terraform state, or Jenkins credentials to GitHub.
- `docker` group membership is effectively root-level access. The source slide's suggestion to run `chmod 777 /var/run/docker.sock` is intentionally not reproduced as an action to follow; see the Jenkins guide.
