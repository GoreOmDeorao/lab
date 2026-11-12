# Terraform — Practical Labs

Commands and HCL examples from Terraform slides 6–29.

## 1. Install and configure AWS CLI + Terraform

Install AWS CLI using the [official Linux instructions](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), then:

```bash
aws --version
```

If the CLI was installed under `~/.local/bin` and is not on `PATH`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
aws --version
```

Configure credentials using an IAM identity intended for the lab:

```bash
aws configure
```

Install Terraform using the [official Terraform installation instructions](https://developer.hashicorp.com/terraform/install), then verify:

```bash
terraform version
```

## 2. Standard Terraform workflow

Run these from the directory containing your `.tf` files:

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

After reviewing resources, type `yes` when prompted by `terraform apply`.

Remove resources managed by the current configuration:

```bash
terraform destroy
```

## 3. Lab 1 — Create a local file

```bash
mkdir -p ~/terraform_1
cd ~/terraform_1
nano main.tf
```

`main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

provider "local" {}

resource "local_file" "example" {
  filename = "hello.txt"
  content  = "Hello Terraform!"
}
```

Run and verify:

```bash
terraform init
terraform plan
terraform apply
ls -l
cat hello.txt
```

Cleanup:

```bash
terraform destroy
```

## 4. Lab 2 — Use variables in `main.tf`

Create `main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
    }
  }
}

provider "local" {}

variable "filename" {
  default = "myfile.txt"
}

variable "content" {
  default = "This is created using variables"
}

resource "local_file" "example" {
  filename = var.filename
  content  = var.content
}
```

Run:

```bash
terraform init
terraform plan
terraform apply
cat myfile.txt
```

## 5. Lab 2.1 — Put variables in separate files

`variables.tf`:

```hcl
variable "filename" {
  description = "Name of the file"
  type        = string
}

variable "content" {
  description = "Content of the file"
  type        = string
}
```

`terraform.tfvars`:

```hcl
filename = "example.txt"
content  = "This file is created using variables from another file"
```

`main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
    }
  }
}

provider "local" {}

resource "local_file" "example" {
  filename = var.filename
  content  = var.content
}
```

Run:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
cat example.txt
```

## 6. Lab 3 — Run NGINX using the Docker provider

Docker Engine must be installed and running on the same machine.

`main.tf`:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

resource "docker_image" "nginx" {
  name = "nginx:latest"
}

resource "docker_container" "nginx_container" {
  name  = "my-nginx"
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = 8080
  }
}
```

Run and test:

```bash
terraform init
terraform plan
terraform apply
docker ps
```

Open `http://localhost:8080`.

> The slide's code maps host port `8080`, but its browser instruction says `localhost:8000`. The correct URL for the shown HCL is `http://localhost:8080`.

Cleanup:

```bash
terraform destroy
```

## 7. Lab 4 — Terraform outputs

`main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
    }
  }
}

provider "local" {}

resource "local_file" "example" {
  filename = "output.txt"
  content  = "This is Terraform output example"
}

output "file_name" {
  value = local_file.example.filename
}

output "file_content" {
  value = local_file.example.content
}
```

Run:

```bash
terraform init
terraform apply
terraform output
terraform output file_name
terraform output file_content
```

## 8. Lab 5 — Create multiple local files

`main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
    }
  }
}

provider "local" {}

resource "local_file" "file1" {
  filename = "file1.txt"
  content  = "First file"
}

resource "local_file" "file2" {
  filename = "file2.txt"
  content  = "Second file"
}
```

Run:

```bash
terraform init
terraform apply
cat file1.txt
cat file2.txt
```

## 9. Lab 6 — Resource dependency using a reference

`main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
    }
  }
}

provider "local" {}

resource "local_file" "file1" {
  filename = "source.txt"
  content  = "This is original file"
}

resource "local_file" "file2" {
  filename = "copy.txt"
  content  = local_file.file1.content
}
```

Run:

```bash
terraform init
terraform apply
cat source.txt
cat copy.txt
```

## 10. Lab 7 — Run an Alpine container

`main.tf`:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

resource "docker_image" "alpine" {
  name = "alpine:latest"
}

resource "docker_container" "alpine_demo" {
  name    = "my-alpine"
  image   = docker_image.alpine.image_id
  command = ["sleep", "300"]
}
```

Run and access:

```bash
terraform init
terraform plan
terraform apply
docker ps
docker exec -it my-alpine sh
exit
```

Cleanup:

```bash
terraform destroy
```

## 11. Lab 8 — Run Apache (`httpd`) container

`main.tf`:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

resource "docker_image" "apache" {
  name = "httpd:latest"
}

resource "docker_container" "apache_demo" {
  name  = "my-apache"
  image = docker_image.apache.image_id

  ports {
    internal = 80
    external = 8081
  }
}
```

Run:

```bash
terraform init
terraform plan
terraform apply
```

Open `http://localhost:8081`.

## 12. Configure AWS credentials

Create an IAM identity for the lab, create access keys if the lab specifically requires them, then configure AWS CLI:

```bash
aws --version
aws configure
aws sts get-caller-identity
```

Do not put credentials in Terraform code or commit `~/.aws/credentials` to GitHub.

## 13. Lab 9 — Create an IAM user and access key

`main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_iam_user" "cdac_user" {
  name = "cdac"
}

resource "aws_iam_access_key" "cdac_key" {
  user = aws_iam_user.cdac_user.name
}
```

Run:

```bash
terraform init
terraform plan
terraform apply
```

> **Sensitive:** Terraform state can contain the generated secret access key. Protect `terraform.tfstate`, do not commit it, and destroy the lab identity when finished. The slide does not include a permissions policy for this IAM user.

## 14. Lab 10 — Create an S3 bucket and upload a file

Create a local file in the Terraform directory:

```bash
echo 'Hello from Terraform' > hello.txt
```

`main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "date_bucket" {
  bucket = "<GLOBALLY_UNIQUE_BUCKET_NAME>"
}

resource "aws_s3_object" "file_upload" {
  bucket = aws_s3_bucket.date_bucket.id
  key    = "hello.txt"
  source = "hello.txt"
}
```

Run:

```bash
terraform init
terraform plan
terraform apply
```

Replace the bucket placeholder with a globally unique lowercase bucket name. Before destroying the bucket, remove any objects if bucket deletion is blocked.

## 15. Lab 11 — Create an EC2 instance

The AMI ID shown on the slide is for an example in `us-east-1`; AMI IDs vary by Region and can become unavailable. The EC2 key pair must already exist in the selected Region.

`main.tf`:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "myec2" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  key_name      = "<YOUR_EXISTING_KEY_PAIR_NAME>"

  tags = {
    Name = "Terraform-EC2"
  }
}
```

Run:

```bash
terraform init
terraform plan
terraform apply
terraform output
```

## 16. Lab 12 — EC2 instance with NGINX

`main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_security_group" "web_sg" {
  name        = "terraform-web-sg"
  description = "Allow HTTP and SSH"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    # Replace with your own public IP in CIDR notation, e.g. 203.0.113.10/32.
    cidr_blocks = ["<YOUR_PUBLIC_IP>/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = "<VALID_AMI_ID_FOR_US_EAST_1>"
  instance_type          = "t2.micro"
  security_groups        = [aws_security_group.web_sg.name]

  user_data = <<-EOF
              #!/bin/bash
              dnf update -y
              dnf install -y nginx
              systemctl enable nginx
              systemctl start nginx
              echo "<h1>Hello from Terraform EC2</h1>" > /usr/share/nginx/html/index.html
              EOF

  tags = {
    Name = "Terraform-Nginx-Server"
  }
}

output "public_ip" {
  value = aws_instance.web.public_ip
}

output "website_url" {
  value = "http://${aws_instance.web.public_ip}"
}
```

Run:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform output public_ip
terraform output website_url
```

> This configuration assumes the default VPC because the source uses the `security_groups` name argument. Use a Region-valid AMI and restrict SSH to your own IP. Do not leave SSH open to `0.0.0.0/0`.

## 17. Lab 13 — EC2 using variables and outputs

`variables.tf`:

```hcl
variable "region" {
  default = "us-east-1"
}

variable "ami_id" {
  default = "<VALID_AMI_ID_FOR_US_EAST_1>"
}

variable "instance_type" {
  default = "t2.micro"
}
```

`main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

resource "aws_instance" "myec2" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name = "Terraform-Variable-EC2"
  }
}
```

`outputs.tf`:

```hcl
output "instance_id" {
  value = aws_instance.myec2.id
}

output "public_ip" {
  value = aws_instance.myec2.public_ip
}
```

Run:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform output
```

Cleanup for AWS labs:

```bash
terraform destroy
```

Source reference: Terraform slides 6–29.
