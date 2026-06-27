## Pre-requisites

- AWS account
- Terraform installed
- Existing AWS Key Pair

---

Run the script provided in this repository on the VM where you want Vault to be installed.

Terraform code to create ec2 instance and run the script

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_security_group" "vault_sg" {
  name        = "vault-security-group"
  description = "Allow SSH and Vault access"

  ingress {
    description = "SSH Access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Vault Port"
    from_port   = 8200
    to_port     = 8200
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "example" {
  ami                    = "ami-091138d0f0d41ff90"
  instance_type          = "t2.micro"
  key_name               = "vault"
  vpc_security_group_ids = [aws_security_group.vault_sg.id]

  user_data = file("${path.module}/script.sh")

  tags = {
    Name = "Vault Demo"
  }
}

output "public_ip" {
  value = aws_instance.example.public_ip
}

If the user_data approach does not work, first create the EC2 instance and then execute the script using Terraform provisioners (file and remote-exec).

This setup installs Vault and performs the Vault + GitHub Actions configuration required to run the ci.yaml workflow automatically on every push to this repository.

Make sure to update:

IP addresses wherever required
AWS Access Key and Secret Key inside script.sh

These credentials are required temporarily so Vault can interact with AWS and complete the initial configuration.



## Why Vault?

Vault is used to store and manage sensitive credentials securely.

### Problem Statement

Consider a scenario where:

- You have Terraform code stored in a GitHub repository
- A `ci.yaml` GitHub Actions workflow deploys infrastructure to AWS
- AWS credentials are stored directly in GitHub Secrets

These AWS credentials are usually **long-lived credentials**.

If:
- The GitHub repository gets compromised, or
- A developer account is compromised

then the AWS account and infrastructure are also at risk.

---

## How Vault Helps

Instead of directly providing permanent AWS credentials to GitHub Actions or Jenkins, Vault acts as a secure intermediary.

Vault integrates with:
- GitHub Actions
- Jenkins
- Kubernetes
- Applications

through authentication mechanisms such as **OIDC**.

### Workflow

1. GitHub Actions authenticates to Vault using OIDC
2. Vault validates the identity/token
3. Vault dynamically creates temporary AWS IAM credentials
4. GitHub Actions uses these temporary credentials to deploy infrastructure
5. Credentials automatically expire after the configured TTL

This removes the need for storing permanent AWS credentials in GitHub or Jenkins.

---

## Benefits

- Eliminates long-lived credentials
- Reduces risk if GitHub/Jenkins is compromised
- Generates temporary IAM users/credentials dynamically
- Credentials expire automatically
- Centralized secret management
- Better security and auditability

---

## Architecture Flow

```text
GitHub Actions / Jenkins
            |
            |  OIDC Authentication
            v
          Vault
            |
            |  Dynamic AWS IAM Credentials
            v
            AWS


Example Use Cases
GitHub Actions + Vault + AWS
GitHub Actions authenticates with Vault
Vault generates temporary AWS credentials
Terraform uses those credentials for infrastructure deployment
Jenkins + Vault
Jenkins fetches secrets dynamically from Vault
No hardcoded credentials inside Jenkins pipelines
Kubernetes + Vault
Applications running inside Kubernetes pods can fetch secrets securely from Vault
Avoids storing secrets inside ConfigMaps or YAML files
Terraform + Vault

Vault can also store:

S3 bucket names
API keys
Database credentials
Tokens
Application secrets

Terraform can retrieve these securely during runtime using Vault providers/configuration.

Security Advantage

Using Vault minimizes the exposure of sensitive credentials across:

GitHub
Jenkins
CI/CD pipelines
Terraform code
Kubernetes workloads

and replaces static credentials with short-lived, dynamically generated credentials.
