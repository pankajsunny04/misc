variables.tf

variable "region" {
  description = "AWS Region"
  type        = string
}

variable "cluster_name" {
  description = "Name of the MSK cluster"
  type        = string
}

variable "kafka_version" {
  description = "Kafka version"
  type        = string
}

variable "broker_node_instance_type" {
  description = "Broker node instance type"
  type        = string
}

variable "number_of_broker_nodes" {
  description = "Number of broker nodes"
  type        = number
}

variable "tags" {
  description = "A map of tags to assign to the resource"
  type        = map(string)
  default     = {}
}

variable "security_module_repo" {
  description = "Git repo for the external security module"
  type        = string
}

variable "s3_bucket_module_repo" {
  description = "Git repo for the external S3 bucket module"
  type        = string
}

variable "authentication_type" {
  description = "Type of authentication (IAM, TLS, BOTH)"
  type        = string
}

variable "use_custom_kms" {
  description = "Boolean to decide whether to use custom KMS"
  type        = bool
  default     = false
}

variable "kms_key_arn" {
  description = "ARN of the custom KMS key"
  type        = string
  default     = ""
}







main.tf

provider "aws" {
  region = var.region
}

module "security" {
  source = var.security_module_repo
}

module "s3_bucket" {
  source = var.s3_bucket_module_repo
}

resource "aws_msk_cluster" "example" {
  cluster_name           = var.cluster_name
  kafka_version          = var.kafka_version
  number_of_broker_nodes = var.number_of_broker_nodes
  broker_node_group_info {
    instance_type = var.broker_node_instance_type
    client_subnets = module.security.subnets
    security_groups = [module.security.security_group]
  }
  encryption_info {
    encryption_at_rest_kms_key_arn = var.use_custom_kms ? var.kms_key_arn : null
    encryption_in_transit {
      client_broker = var.authentication_type == "TLS" || var.authentication_type == "BOTH" ? "TLS" : "TLS_PLAINTEXT"
      in_cluster    = true
    }
  }
  configuration_info {
    arn = aws_msk_configuration.example.arn
    revision = aws_msk_configuration.example.latest_revision
  }
  tags = var.tags
}

resource "aws_msk_configuration" "example" {
  name          = "example-configuration"
  kafka_versions = [var.kafka_version]
  server_properties = <<PROPERTIES
auto.create.topics.enable = true
delete.topic.enable = true
PROPERTIES
}

resource "aws_acm_certificate" "private_certificate" {
  count = var.authentication_type == "TLS" || var.authentication_type == "BOTH" ? 1 : 0

  domain_name       = "example.com"
  validation_method = "DNS"
  private_key       = file("${path.module}/private-key.pem")
  certificate_body  = file("${path.module}/certificate.pem")
  certificate_chain = file("${path.module}/chain.pem")
}

resource "aws_iam_role" "msk_iam_role" {
  count = var.authentication_type == "IAM" || var.authentication_type == "BOTH" ? 1 : 0

  name = "msk-iam-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "kafka.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
  tags = var.tags
}




outputs.tf
output "msk_cluster_arn" {
  value = aws_msk_cluster.example.arn
}

output "security_group_id" {
  value = module.security.security_group_id
}

output "s3_bucket_name" {
  value = module.s3_bucket.bucket_name
}




README.md
# AWS MSK Terraform Module

This module creates an AWS MSK (Managed Streaming for Kafka) cluster with the following features:
- Everything is configurable via variables.
- Multiple tag options.
- Integration with external security and S3 bucket modules from a git repo.
- Support for IAM, TLS, or both authentication methods. Generates AWS private certificate for TLS.
- Option to use custom KMS key.

## Usage

```hcl
module "msk" {
  source                 = "./path_to_your_module"
  region                 = "us-west-2"
  cluster_name           = "example-cluster"
  kafka_version          = "2.6.0"
  broker_node_instance_type = "kafka.m5.large"
  number_of_broker_nodes = 3
  tags                   = {
    Environment = "dev"
    Project     = "example"
  }
  security_module_repo   = "git::https://github.com/your/repo.git//path/to/security_module"
  s3_bucket_module_repo  = "git::https://github.com/your/repo.git//path/to/s3_bucket_module"
  authentication_type    = "TLS"
  use_custom_kms         = true
  kms_key_arn            = "arn:aws:kms:us-west-2:123456789012:key/your-key-id"
}






**********************************************************************************
