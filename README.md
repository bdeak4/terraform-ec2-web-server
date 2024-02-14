# terraform-module-ec2-web-server

Setup web server on AWS EC2

## How to use

```tf
terraform {
  required_version = ">= 1.0.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }

    sops = {
      source  = "carlpett/sops"
      version = "~> 0.5"
    }
  }

  backend "s3" {
    bucket         = "project-tfstate"
    dynamodb_table = "project-tfstate-lock"
    region         = "us-east-1"
    profile        = "project"
    encrypt        = true
  }
}

provider "aws" {
  region  = "eu-central-1"
  profile = "project"
}

provider "cloudflare" {
  api_token = data.sops_file.secrets.data["cloudflare_api_token"]
}

data "cloudflare_zone" "domain" {
  name = "project.com"
}

data "sops_file" "secrets" {
  source_file = "secrets.enc.json"
}

module "api" {
  source = "github.com/bdeak4/terraform-ec2-web-server"

  name_prefix               = "project-api-production"
  instance_type             = "t3a.small"
  instance_count            = 1
  instance_root_device_size = 20
  subnets                   = data.aws_subnets.public_subnets.ids
  security_groups           = data.aws_security_groups.public_sg.ids
  ssh_public_key            = file("../../../ssh-keys/production.pub")
  website_domain            = "api.project.com"
  cloudflare_zone_id        = data.cloudflare_zone.domain.id

  tags = {
    Project     = "project"
    Role        = "api"
    Environment = "production"
  }
}
```