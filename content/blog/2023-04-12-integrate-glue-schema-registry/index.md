---
title: Integrate Glue Schema Registry with Your Python Kafka App
date: 2023-04-12
draft: false
featured: false
draft: false
comment: true
toc: true
reward: false
pinned: false
carousel: false
featuredImage: false
# series:
#   - API development with R
categories:
  - Data Engineering
tags: 
  - AWS
  - Amazon MSK
  - AWS Glue Schema Registry
  - AWS Lambda
  - AWS Serverless Application Model
  - Kafka
  - Python
  - Docker
  - Docker Compose
  - Terraform
authors:
  - JaehyeonKim
images: []
cevo: 26
---

As Kafka producer and consumer apps are decoupled, they operate on Kafka topics rather than communicating with each other directly. As described in the [Confluent document](https://docs.confluent.io/platform/current/schema-registry/index.html#sr-overview), _Schema Registry_ provides a centralized repository for managing and validating schemas for topic message data, and for serialization and deserilazation of the data over the network. Producers and consumers to Kafka topics can use schemas to ensure data consistency and compatibility as schemas evolve. In AWS, the [Glue Schema Registry](https://docs.aws.amazon.com/glue/latest/dg/schema-registry.html) supports features to manage and enforce schemas on data streaming applications using convenient integrations with Apache Kafka, [Amazon Managed Streaming for Apache Kafka](https://aws.amazon.com/msk/), [Amazon Kinesis Data Streams](https://aws.amazon.com/kinesis/data-streams/), [Amazon Kinesis Data Analytics for Apache Flink](https://aws.amazon.com/kinesis/data-analytics/), and [AWS Lambda](https://aws.amazon.com/lambda/). 

While the Glue Schema Registry facilitate Kafka application development, most example implementations target JVM languages, and it is not easy to find a Python example. In this post, we will discuss how to integrate Python Kafka producer and consumer apps with the Glue Schema Registry.

## Architecture

![](featured.png#center)

## Infrastructure
A VPC with 3 public and private subnets is created using the [AWS VPC Terraform module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) (`vpc.tf`). Also, a [SoftEther VPN](https://www.softether.org/) server is deployed in order to access the resources in the private subnets from the developer machine (`vpn.tf`). It is particularly useful to monitor and manage the MSK cluster and Kafka topic as well as developing the Kafka producer Lambda function locally. The details about how to configure the VPN server can be found in an [earlier post](https://cevo.com.au/post/simplify-your-development-on-aws-with-terraform/). The source can be found in the [**GitHub repository**](https://github.com/jaehyeon-kim/kafka-pocs/tree/main/glue-schema-registry) of this post.

### MSK
A MSK cluster with 3 brokers is created. The broker nodes are deployed with the `kafka.m5.large` instance type in the private subnets. IAM authentication is used for the client authentication method. Note this method is the only secured authentication method supported by Redshift because the external schema supports either the no authentication or IAM authentication method only.

```terraform
# glue-schema-registry/infra/variable.tf
locals {
  ...
  msk = {
    version          = "3.3.1"
    instance_size    = "kafka.m5.large"
    ebs_volume_size  = 20
    log_retention_ms = 604800000 # 7 days
  }
  ...
}
# glue-schema-registry/infra/msk.tf
resource "aws_msk_cluster" "msk_data_cluster" {
  cluster_name           = "${local.name}-msk-cluster"
  kafka_version          = local.msk.version
  number_of_broker_nodes = length(module.vpc.private_subnets)
  configuration_info {
    arn      = aws_msk_configuration.msk_config.arn
    revision = aws_msk_configuration.msk_config.latest_revision
  }

  broker_node_group_info {
    instance_type   = local.msk.instance_size
    client_subnets  = module.vpc.private_subnets
    security_groups = [aws_security_group.msk.id]
    storage_info {
      ebs_storage_info {
        volume_size = local.msk.ebs_volume_size
      }
    }
  }

  client_authentication {
    sasl {
      iam = true
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk_cluster_lg.name
      }
      s3 {
        enabled = true
        bucket  = aws_s3_bucket.default_bucket.id
        prefix  = "logs/msk/cluster-"
      }
    }
  }

  tags = local.tags

  depends_on = [aws_msk_configuration.msk_config]
}

resource "aws_msk_configuration" "msk_config" {
  name = "${local.name}-msk-configuration"

  kafka_versions = [local.msk.version]

  server_properties = <<PROPERTIES
    auto.create.topics.enable = true
    delete.topic.enable = true
    log.retention.ms = ${local.msk.log_retention_ms}
  PROPERTIES
}

resource "aws_security_group" "msk" {
  name   = "${local.name}-msk-sg"
  vpc_id = module.vpc.vpc_id

  lifecycle {
    create_before_destroy = true
  }

  tags = local.tags
}
```

#### Security Groups
Two security groups are created - one for the MSK cluster and the other for the Lambda apps. 

The inbound/outbound rules of the former are created for accessing the cluster by
* Event Source Mapping (ESM) for Lambda
  * This is for the Lambda consumer that subscribes the MSK cluster. As described in the [AWS re:Post doc](https://repost.aws/knowledge-center/lambda-trigger-msk-kafka-cluster), when a Lambda function is configured with an Amazon MSK trigger or a self-managed Kafka trigger, an ESM resource is automatically created. An ESM is separate from the Lambda function, and it continuously polls records from the topic in the Kafka cluster. The ESM bundles those records into a payload. Then, it calls the Lambda Invoke API to deliver the payload to your Lambda function for processing. Note it doesn't inherit the VPC network settings of the Lambda function but uses the subnet and security group settings that are configured on the target MSK cluster. Therefore, the MSK cluster's security group must include a rule that grants ingress traffic from itself and egress traffic to itself. For us, the rules on port 9098 as the cluster only supports IAM authentication. Also, an additional egress rule is created to access the Glue Schema Registry.
* Other Resources
  * Two ingress rules are created for the VPN server and Lambda. The latter is only for the Lambda producer because the consumer doesn't rely on the Lambda network setting.

The second security group is created here, while the Lambda function is created in a different Terraform stack. This is for ease of adding it to the inbound rule of the MSK's security group. Later we will discuss how to make use of it with the Lambda function. The outbound rule allows all outbound traffic although only port 9098 for the MSK cluster and 443 for the Glue Schema Registry would be sufficient.

```terraform
# glue-schema-registry/infra/msk.tf
resource "aws_security_group" "msk" {
  name   = "${local.name}-msk-sg"
  vpc_id = module.vpc.vpc_id

  lifecycle {
    create_before_destroy = true
  }

  tags = local.tags
}

# for lambda event source mapping
resource "aws_security_group_rule" "msk_ingress_self_broker" {
  type                     = "ingress"
  description              = "msk ingress self"
  security_group_id        = aws_security_group.msk.id
  protocol                 = "tcp"
  from_port                = 9098
  to_port                  = 9098
  source_security_group_id = aws_security_group.msk.id
}

resource "aws_security_group_rule" "msk_egress_self_broker" {
  type                     = "egress"
  description              = "msk egress self"
  security_group_id        = aws_security_group.msk.id
  protocol                 = "tcp"
  from_port                = 9098
  to_port                  = 9098
  source_security_group_id = aws_security_group.msk.id
}

resource "aws_security_group_rule" "msk_all_outbound" {
  type              = "egress"
  description       = "allow outbound all"
  security_group_id = aws_security_group.msk.id
  protocol          = "-1"
  from_port         = "0"
  to_port           = "0"
  cidr_blocks       = ["0.0.0.0/0"]
}

# for other resources
resource "aws_security_group_rule" "msk_vpn_inbound" {
  count                    = local.vpn.to_create ? 1 : 0
  type                     = "ingress"
  description              = "VPN access"
  security_group_id        = aws_security_group.msk.id
  protocol                 = "tcp"
  from_port                = 9098
  to_port                  = 9098
  source_security_group_id = aws_security_group.vpn[0].id
}

resource "aws_security_group_rule" "msk_lambda_inbound" {
  type                     = "ingress"
  description              = "lambda access"
  security_group_id        = aws_security_group.msk.id
  protocol                 = "tcp"
  from_port                = 9098
  to_port                  = 9098
  source_security_group_id = aws_security_group.kafka_app_lambda.id
}

...

# lambda security group
resource "aws_security_group" "kafka_app_lambda" {
  name   = "${local.name}-lambda-msk-access"
  vpc_id = module.vpc.vpc_id

  lifecycle {
    create_before_destroy = true
  }

  tags = local.tags
}

resource "aws_security_group_rule" "kafka_app_lambda_msk_egress" {
  type              = "egress"
  description       = "allow outbound all"
  security_group_id = aws_security_group.kafka_app_lambda.id
  protocol          = "-1"
  from_port         = 0
  to_port           = 0
  cidr_blocks       = ["0.0.0.0/0"]
}
```

## Kafka Apps
The resources related to the Kafka producer and consumer Lambda functions are managed in a separate Terraform stack. This is because it is easier to build the relevant resources iteratively. Note the SAM CLI builds the whole Terraform stack even for a small change of code, and it wouldn’t be convenient if the entire resources are managed in the same stack.

### Producer
dd
#### Lambda Function
dd
#### IAM Permission
dd

### EventBridge Rule

## Kafka Consumer

### Consumer Source

### Lambda Function

#### IAM Permission

## Schema Evolution Demo

## Deployment

### Topic Creation

### Local Testing with SAM

###