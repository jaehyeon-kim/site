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

As Kafka producer and consumer apps are decoupled, they operate on Kafka topics rather than communicating with each other directly. As described in the [Confluent document](https://docs.confluent.io/platform/current/schema-registry/index.html#sr-overview), _Schema Registry_ provides a centralized repository for managing and validating schemas for topic message data, and for serialization and deserialization of the data over the network. Producers and consumers to Kafka topics can use schemas to ensure data consistency and compatibility as schemas evolve. In AWS, the [Glue Schema Registry](https://docs.aws.amazon.com/glue/latest/dg/schema-registry.html) supports features to manage and enforce schemas on data streaming applications using convenient integrations with Apache Kafka, [Amazon Managed Streaming for Apache Kafka](https://aws.amazon.com/msk/), [Amazon Kinesis Data Streams](https://aws.amazon.com/kinesis/data-streams/), [Amazon Kinesis Data Analytics for Apache Flink](https://aws.amazon.com/kinesis/data-analytics/), and [AWS Lambda](https://aws.amazon.com/lambda/). 

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

#### Order Data

```python
# glue-schema-registry/app/producer/src/order.py
import datetime
import string
import json
import typing
import dataclasses
import enum

from faker import Faker
from dataclasses_avroschema import AvroModel


class Compatibility(enum.Enum):
    NONE = "NONE"
    DISABLED = "DISABLED"
    BACKWARD = "BACKWARD"
    BACKWARD_ALL = "BACKWARD_ALL"
    FORWARD = "FORWARD"
    FORWARD_ALL = "FORWARD_ALL"
    FULL = "FULL"
    FULL_ALL = "FULL_ALL"


class InjectCompatMixin:
    @classmethod
    def updated_avro_schema_to_python(cls, compat: Compatibility = Compatibility.BACKWARD):
        schema = cls.avro_schema_to_python()
        schema["compatibility"] = compat.value
        return schema

    @classmethod
    def updated_avro_schema(cls, compat: Compatibility = Compatibility.BACKWARD):
        schema = cls.updated_avro_schema_to_python(compat)
        return json.dumps(schema)


@dataclasses.dataclass
class OrderItem(AvroModel):
    product_id: int
    quantity: int


@dataclasses.dataclass
class Order(AvroModel, InjectCompatMixin):
    "Online fake order item"
    order_id: str
    ordered_at: datetime.datetime
    user_id: str
    order_items: typing.List[OrderItem]

    class Meta:
        namespace = "Order V1"

    def asdict(self):
        return dataclasses.asdict(self)

    @classmethod
    def auto(cls, fake: Faker = Faker()):
        rand_int = fake.random_int(1, 1000)
        user_id = "".join(
            [string.ascii_lowercase[int(s)] if s.isdigit() else s for s in hex(rand_int)]
        )[::-1]
        order_items = [
            OrderItem(fake.random_int(1, 9999), fake.random_int(1, 10))
            for _ in range(fake.random_int(1, 4))
        ]
        return cls(fake.uuid4(), datetime.datetime.utcnow(), user_id, order_items)

    def create(self, num: int):
        return [self.auto() for _ in range(num)]


@dataclasses.dataclass
class OrderMore(Order):
    is_prime: bool

    @classmethod
    def auto(cls, fake: Faker = Faker()):
        o = Order.auto()
        return cls(o.order_id, o.ordered_at, o.user_id, o.order_items, fake.pybool())
```

```json
{
	"doc": "Online fake order item",
	"namespace": "Order V1",
	"name": "Order",
	"compatibility": "BACKWARD",
	"type": "record",
	"fields": [
		{
			"name": "order_id",
			"type": "string"
		},
		{
			"name": "ordered_at",
			"type": {
				"type": "long",
				"logicalType": "timestamp-millis"
			}
		},
		{
			"name": "user_id",
			"type": "string"
		},
		{
			"name": "order_items",
			"type": {
				"type": "array",
				"items": {
					"type": "record",
					"name": "OrderItem",
					"fields": [
						{
							"name": "product_id",
							"type": "long"
						},
						{
							"name": "quantity",
							"type": "long"
						}
					]
				},
				"name": "order_item"
			}
		}
	]
}
```

```json
{
  "order_id": "53263c42-81b3-4a53-8067-fcdb44fa5479",
  "ordered_at": 1680745813045,
  "user_id": "dicxa",
  "order_items": [
    {
      "product_id": 5947,
      "quantity": 8
    }
  ]
}
```


#### Producer

```python
# glue-schema-registry/app/producer/src/producer.py
import os
import datetime
import json
import typing

import boto3
import botocore.exceptions
from kafka import KafkaProducer
from aws_schema_registry import SchemaRegistryClient
from aws_schema_registry.avro import AvroSchema
from aws_schema_registry.adapter.kafka import KafkaSerializer
from aws_schema_registry.exception import SchemaRegistryException

from .order import Order


class Producer:
    def __init__(self, bootstrap_servers: list, topic: str, registry: str, is_local: bool = False):
        self.bootstrap_servers = bootstrap_servers
        self.topic = topic
        self.registry = registry
        self.glue_client = boto3.client(
            "glue", region_name=os.getenv("AWS_DEFAULT_REGION", "ap-southeast-2")
        )
        self.is_local = is_local
        self.producer = self.create()

    @property
    def serializer(self):
        client = SchemaRegistryClient(self.glue_client, registry_name=self.registry)
        return KafkaSerializer(client)

    def create(self):
        params = {
            "bootstrap_servers": self.bootstrap_servers,
            "key_serializer": lambda v: json.dumps(v, default=self.serialize).encode("utf-8"),
            "value_serializer": self.serializer,
        }
        if not self.is_local:
            params = {
                **params,
                **{"security_protocol": "SASL_SSL", "sasl_mechanism": "AWS_MSK_IAM"},
            }
        return KafkaProducer(**params)

    def send(self, orders: typing.List[Order], schema: AvroSchema):
        if not self.check_registry():
            print(f"registry not found, create {self.registry}")
            self.create_registry()

        for order in orders:
            data = order.asdict()
            try:
                self.producer.send(
                    self.topic, key={"order_id": data["order_id"]}, value=(data, schema)
                )
            except SchemaRegistryException as e:
                raise RuntimeError("fails to send a message") from e
        self.producer.flush()

    def serialize(self, obj):
        if isinstance(obj, datetime.datetime):
            return obj.isoformat()
        if isinstance(obj, datetime.date):
            return str(obj)
        return obj

    def check_registry(self):
        try:
            self.glue_client.get_registry(RegistryId={"RegistryName": self.registry})
            return True
        except botocore.exceptions.ClientError as e:
            if e.response["Error"]["Code"] == "EntityNotFoundException":
                return False
            else:
                raise e

    def create_registry(self):
        try:
            self.glue_client.create_registry(RegistryName=self.registry)
            return True
        except botocore.exceptions.ClientError as e:
            if e.response["Error"]["Code"] == "AlreadyExistsException":
                return True
            else:
                raise e
```

#### Lambda Handler

```python
# glue-schema-registry/app/producer/lambda_handler.py
import os
import datetime
import time

from aws_schema_registry.avro import AvroSchema

from src.order import Order, OrderMore, Compatibility
from src.producer import Producer


def lambda_function(event, context):
    producer = Producer(
        bootstrap_servers=os.environ["BOOTSTRAP_SERVERS"].split(","),
        topic=os.environ["TOPIC_NAME"],
        registry=os.environ["REGISTRY_NAME"],
    )
    s = datetime.datetime.now()
    ttl_rec = 0
    while True:
        orders = Order.auto().create(100)
        schema = AvroSchema(Order.updated_avro_schema(Compatibility.BACKWARD))
        producer.send(orders, schema)
        ttl_rec += len(orders)
        print(f"sent {len(orders)} messages")
        elapsed_sec = (datetime.datetime.now() - s).seconds
        if elapsed_sec > int(os.getenv("MAX_RUN_SEC", "60")):
            print(f"{ttl_rec} records are sent in {elapsed_sec} seconds ...")
            break
        time.sleep(1)


if __name__ == "__main__":
    producer = Producer(
        bootstrap_servers=os.environ["BOOTSTRAP_SERVERS"].split(","),
        topic=os.environ["TOPIC_NAME"],
        registry=os.environ["REGISTRY_NAME"],
        is_local=True,
    )
    use_more = os.getenv("USE_MORE") is not None
    if not use_more:
        orders = Order.auto().create(1)
        schema = AvroSchema(Order.updated_avro_schema(Compatibility.BACKWARD))
    else:
        orders = OrderMore.auto().create(1)
        schema = AvroSchema(OrderMore.updated_avro_schema(Compatibility.BACKWARD))
    print(orders)
    producer.send(orders, schema)
```

#### Lambda Resource

```terraform
# glue-schema-registry/app/variables.tf
locals {
  name        = local.infra_prefix
  region      = data.aws_region.current.name
  environment = "dev"

  infra_prefix = "glue-schema-registry"

  producer = {
    src_path          = "producer"
    function_name     = "kafka_producer"
    handler           = "lambda_handler.lambda_function"
    concurrency       = 5
    timeout           = 90
    memory_size       = 128
    runtime           = "python3.8"
    schedule_rate     = "rate(1 minute)"
    to_enable_trigger = true
    environment = {
      topic_name    = "orders"
      registry_name = "customer"
      max_run_sec   = 60
    }
  }

  ...
}

# glue-schema-registry/app/main.tf
module "kafka_producer_lambda" {
  source = "terraform-aws-modules/lambda/aws"

  function_name          = local.producer.function_name
  handler                = local.producer.handler
  runtime                = local.producer.runtime
  timeout                = local.producer.timeout
  memory_size            = local.producer.memory_size
  source_path            = local.producer.src_path
  vpc_subnet_ids         = data.aws_subnets.private.ids
  vpc_security_group_ids = [data.aws_security_group.kafka_app_lambda.id]
  attach_network_policy  = true
  attach_policies        = true
  policies               = [aws_iam_policy.msk_lambda_producer_permission.arn]
  number_of_policies     = 1
  environment_variables = {
    BOOTSTRAP_SERVERS = data.aws_msk_cluster.msk_data_cluster.bootstrap_brokers_sasl_iam
    TOPIC_NAME        = local.producer.environment.topic_name
    REGISTRY_NAME     = local.producer.environment.registry_name
    MAX_RUN_SEC       = local.producer.environment.max_run_sec
  }

  tags = local.tags
}

resource "aws_lambda_function_event_invoke_config" "kafka_producer_lambda" {
  function_name          = module.kafka_producer_lambda.lambda_function_name
  maximum_retry_attempts = 0
}

resource "aws_lambda_permission" "allow_eventbridge" {
  count         = local.producer.to_enable_trigger ? 1 : 0
  statement_id  = "InvokeLambdaFunction"
  action        = "lambda:InvokeFunction"
  function_name = local.producer.function_name
  principal     = "events.amazonaws.com"
  source_arn    = module.eventbridge.eventbridge_rule_arns["crons"]

  depends_on = [
    module.eventbridge
  ]
}
```

##### IAM Permission

```terraform
# glue-schema-registry/app/main.tf
resource "aws_iam_policy" "msk_lambda_producer_permission" {
  name = "${local.producer.function_name}-msk-lambda-producer-permission"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid = "PermissionOnCluster"
        Action = [
          "kafka-cluster:Connect",
          "kafka-cluster:AlterCluster",
          "kafka-cluster:DescribeCluster"
        ]
        Effect   = "Allow"
        Resource = "arn:aws:kafka:${local.region}:${data.aws_caller_identity.current.account_id}:cluster/${local.infra_prefix}-msk-cluster/*"
      },
      {
        Sid = "PermissionOnTopics"
        Action = [
          "kafka-cluster:*Topic*",
          "kafka-cluster:WriteData",
          "kafka-cluster:ReadData"
        ]
        Effect   = "Allow"
        Resource = "arn:aws:kafka:${local.region}:${data.aws_caller_identity.current.account_id}:topic/${local.infra_prefix}-msk-cluster/*"
      },
      {
        Sid = "PermissionOnGroups"
        Action = [
          "kafka-cluster:AlterGroup",
          "kafka-cluster:DescribeGroup"
        ]
        Effect   = "Allow"
        Resource = "arn:aws:kafka:${local.region}:${data.aws_caller_identity.current.account_id}:group/${local.infra_prefix}-msk-cluster/*"
      },
      {
        Sid = "PermissionOnGlueSchema"
        Action = [
          "glue:*Schema*",
          "glue:GetRegistry",
          "glue:CreateRegistry",
          "glue:ListRegistries",
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}
```

#### EventBridge Rule

```terraform
module "eventbridge" {
  source = "terraform-aws-modules/eventbridge/aws"

  create_bus = false

  rules = {
    crons = {
      description         = "Kafka producer lambda schedule"
      schedule_expression = local.producer.schedule_rate
    }
  }

  targets = {
    crons = [for i in range(local.producer.concurrency) : {
      name = "lambda-target-${i}"
      arn  = module.kafka_producer_lambda.lambda_function_arn
    }]
  }

  depends_on = [
    module.kafka_producer_lambda
  ]

  tags = local.tags
}
```

### Consumer

#### Lambda Handler

```python
# glue-schema-registry/app/consumer/lambda_handler.py
import os
import json
import base64
import datetime

import boto3
from aws_schema_registry import SchemaRegistryClient
from aws_schema_registry.adapter.kafka import Deserializer, KafkaDeserializer


class LambdaDeserializer(Deserializer):
    def __init__(self, registry: str):
        self.registry = registry

    @property
    def deserializer(self):
        glue_client = boto3.client(
            "glue", region_name=os.getenv("AWS_DEFAULT_REGION", "ap-southeast-2")
        )
        client = SchemaRegistryClient(glue_client, registry_name=self.registry)
        return KafkaDeserializer(client)

    def deserialize(self, topic: str, bytes_: bytes):
        return self.deserializer.deserialize(topic, bytes_)


class ConsumerRecord:
    def __init__(self, record: dict):
        self.topic = record["topic"]
        self.partition = record["partition"]
        self.offset = record["offset"]
        self.timestamp = record["timestamp"]
        self.timestamp_type = record["timestampType"]
        self.key = record["key"]
        self.value = record["value"]
        self.headers = record["headers"]

    def parse_key(self):
        return base64.b64decode(self.key).decode()

    def parse_value(self, deserializer: LambdaDeserializer):
        parsed = deserializer.deserialize(self.topic, base64.b64decode(self.value))
        return parsed.data

    def format_timestamp(self, to_str: bool = True):
        ts = datetime.datetime.fromtimestamp(self.timestamp / 1000)
        if to_str:
            return ts.isoformat()
        return ts

    def parse_record(
        self, deserializer: LambdaDeserializer, to_str: bool = True, to_json: bool = True
    ):
        rec = {
            **self.__dict__,
            **{
                "key": self.parse_key(),
                "value": self.parse_value(deserializer),
                "timestamp": self.format_timestamp(to_str),
            },
        }
        if to_json:
            return json.dumps(rec, default=self.serialize)
        return rec

    def serialize(self, obj):
        if isinstance(obj, datetime.datetime):
            return obj.isoformat()
        if isinstance(obj, datetime.date):
            return str(obj)
        return obj


def lambda_function(event, context):
    deserializer = LambdaDeserializer(os.getenv("REGISTRY_NAME", "customer"))
    for _, records in event["records"].items():
        for record in records:
            cr = ConsumerRecord(record)
            print(cr.parse_record(deserializer))
```

#### Lambda Resource

```terraform
# glue-schema-registry/app/variables.tf
locals {
  name        = local.infra_prefix
  region      = data.aws_region.current.name
  environment = "dev"

  infra_prefix = "glue-schema-registry"

  ...

  consumer = {
    src_path          = "consumer"
    function_name     = "kafka_consumer"
    handler           = "lambda_handler.lambda_function"
    timeout           = 90
    memory_size       = 128
    runtime           = "python3.8"
    topic_name        = "orders"
    starting_position = "TRIM_HORIZON"
    environment = {
      registry_name = "customer"
    }
  }
}

# glue-schema-registry/app/main.tf
module "kafka_consumer_lambda" {
  source = "terraform-aws-modules/lambda/aws"

  function_name          = local.consumer.function_name
  handler                = local.consumer.handler
  runtime                = local.consumer.runtime
  timeout                = local.consumer.timeout
  memory_size            = local.consumer.memory_size
  source_path            = local.consumer.src_path
  vpc_subnet_ids         = data.aws_subnets.private.ids
  vpc_security_group_ids = [data.aws_security_group.kafka_app_lambda.id]
  attach_network_policy  = true
  attach_policies        = true
  policies               = [aws_iam_policy.msk_lambda_consumer_permission.arn]
  number_of_policies     = 1
  environment_variables = {
    REGISTRY_NAME = local.producer.environment.registry_name
  }

  tags = local.tags
}

resource "aws_lambda_event_source_mapping" "kafka_consumer_lambda" {
  event_source_arn  = data.aws_msk_cluster.msk_data_cluster.arn
  function_name     = module.kafka_consumer_lambda.lambda_function_name
  topics            = [local.consumer.topic_name]
  starting_position = local.consumer.starting_position
  amazon_managed_kafka_event_source_config {
    consumer_group_id = "${local.consumer.topic_name}-group-01"
  }
}

resource "aws_lambda_permission" "allow_msk" {
  statement_id  = "InvokeLambdaFunction"
  action        = "lambda:InvokeFunction"
  function_name = local.consumer.function_name
  principal     = "kafka.amazonaws.com"
  source_arn    = data.aws_msk_cluster.msk_data_cluster.arn
}
```

##### IAM Permission

```terraform
# glue-schema-registry/app/main.tf
resource "aws_iam_policy" "msk_lambda_consumer_permission" {
  name = "${local.consumer.function_name}-msk-lambda-consumer-permission"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid = "PermissionOnKafkaCluster"
        Action = [
          "kafka-cluster:Connect",
          "kafka-cluster:DescribeGroup",
          "kafka-cluster:AlterGroup",
          "kafka-cluster:DescribeTopic",
          "kafka-cluster:ReadData",
          "kafka-cluster:DescribeClusterDynamicConfiguration"
        ]
        Effect = "Allow"
        Resource = [
          "arn:aws:kafka:${local.region}:${data.aws_caller_identity.current.account_id}:cluster/${local.infra_prefix}-msk-cluster/*",
          "arn:aws:kafka:${local.region}:${data.aws_caller_identity.current.account_id}:topic/${local.infra_prefix}-msk-cluster/*",
          "arn:aws:kafka:${local.region}:${data.aws_caller_identity.current.account_id}:group/${local.infra_prefix}-msk-cluster/*"
        ]
      },
      {
        Sid = "PermissionOnKafka"
        Action = [
          "kafka:DescribeCluster",
          "kafka:GetBootstrapBrokers"
        ]
        Effect   = "Allow"
        Resource = "*"
      },
      {
        Sid = "PermissionOnNetwork"
        Action = [
          # The first three actions also exist in netwrok policy attachment in lambda module
          # "ec2:CreateNetworkInterface",
          # "ec2:DescribeNetworkInterfaces",
          # "ec2:DeleteNetworkInterface",
          "ec2:DescribeVpcs",
          "ec2:DescribeSubnets",
          "ec2:DescribeSecurityGroups"
        ]
        Effect   = "Allow"
        Resource = "*"
      },
      {
        Sid = "PermissionOnGlueSchema"
        Action = [
          "glue:*Schema*",
          "glue:ListRegistries"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}
```
## Schema Evolution Demo

```yaml
# glue-schema-registry/compose-local.yml
version: "3.5"

services:
  zookeeper:
    image: docker.io/bitnami/zookeeper:3.8
    container_name: zookeeper
    ports:
      - "2181"
    networks:
      - kafkanet
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    volumes:
      - zookeeper_data:/bitnami/zookeeper
  kafka-0:
    image: docker.io/bitnami/kafka:3.3
    container_name: kafka-0
    expose:
      - 9092
    ports:
      - "9093:9093"
    networks:
      - kafkanet
    environment:
      - ALLOW_PLAINTEXT_LISTENER=yes
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_CFG_BROKER_ID=0
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CLIENT:PLAINTEXT,EXTERNAL:PLAINTEXT
      - KAFKA_CFG_LISTENERS=CLIENT://:9092,EXTERNAL://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=CLIENT://kafka-0:9092,EXTERNAL://localhost:9093
      - KAFKA_INTER_BROKER_LISTENER_NAME=CLIENT
    volumes:
      - kafka_0_data:/bitnami/kafka
    depends_on:
      - zookeeper
  kpow:
    image: factorhouse/kpow-ce:91.2.1
    container_name: kpow
    ports:
      - "3000:3000"
    networks:
      - kafkanet
    environment:
      AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY
      AWS_SESSION_TOKEN: $AWS_SESSION_TOKEN
      BOOTSTRAP: kafka-0:9092
      SCHEMA_REGISTRY_ARN: $SCHEMA_REGISTRY_ARN
      SCHEMA_REGISTRY_REGION: $AWS_DEFAULT_REGION
    depends_on:
      - zookeeper
      - kafka-0

networks:
  kafkanet:
    name: kafka-network

volumes:
  zookeeper_data:
    driver: local
  kafka_0_data:
    driver: local
```

```bash
export BOOTSTRAP_SERVERS=localhost:9093
export TOPIC_NAME=demo
export REGISTRY_NAME=customer

cd glue-schema-registry/app/producer

## initially send an order record and register schema
python lambda_handler.py

## attempted to create a new record with unsupported schema
export USE_MORE=1

python lambda_handler.py
```

`aws_schema_registry.exception.SchemaRegistryException: Schema Found but status is FAILURE`

![](schema-failure.png#center)


## Deployment

```yaml
# glue-schema-registry/docker-compose.yml
version: "3.5"

services:
  kpow:
    image: factorhouse/kpow-ce:91.2.1
    container_name: kpow
    ports:
      - "3000:3000"
    networks:
      - kafkanet
    environment:
      AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY
      AWS_SESSION_TOKEN: $AWS_SESSION_TOKEN
      # kafka cluster
      BOOTSTRAP: $BOOTSTRAP_SERVERS
      SECURITY_PROTOCOL: SASL_SSL
      SASL_MECHANISM: AWS_MSK_IAM
      SASL_CLIENT_CALLBACK_HANDLER_CLASS: software.amazon.msk.auth.iam.IAMClientCallbackHandler
      SASL_JAAS_CONFIG: software.amazon.msk.auth.iam.IAMLoginModule required;
      # msk connect
      CONNECT_AWS_REGION: $AWS_DEFAULT_REGION
      # glue schema registry
      SCHEMA_REGISTRY_ARN: $SCHEMA_REGISTRY_ARN
      SCHEMA_REGISTRY_REGION: $AWS_DEFAULT_REGION

networks:
  kafkanet:
    name: kafka-network
```

### Topic Creation

![](kpow-create-topic-01.png#center)

dd

![](kpow-create-topic-02.png#center)

### Local Testing with SAM

```terraform
# glue-schema-registry/app/variables.tf
locals {
  ...

  producer = {
    ...
    to_enable_trigger = false
    environment = {
      topic_name    = "orders"
      registry_name = "customer"
      max_run_sec   = 10
    }
  }
  ...
}
```

```bash
$ sam build --hook-name terraform --beta-features

# Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

# Build Succeeded

# Built Artifacts  : .aws-sam/build
# Built Template   : .aws-sam/build/template.yaml

# Commands you can use next
# =========================
# [*] Invoke Function: sam local invoke --hook-name terraform
# [*] Emulate local Lambda functions: sam local start-lambda --hook-name terraform

# SAM CLI update available (1.78.0); (1.70.0 installed)
# To download: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html
```

```bash
$ sam local invoke --hook-name terraform module.kafka_producer_lambda.aws_lambda_function.this[0] --beta-features

# Experimental features are enabled for this session.
# Visit the docs page to learn more about the AWS Beta terms https://aws.amazon.com/service-terms/.

# Skipped prepare hook. Current application is already prepared.
# Invoking lambda_handler.lambda_function (python3.8)
# Skip pulling image and use local one: public.ecr.aws/sam/emulation-python3.8:rapid-1.70.0-x86_64.

# Mounting /home/jaehyeon/personal/kafka-pocs/glue-schema-registry/app/.aws-sam/build/ModuleKafkaProducerLambdaAwsLambdaFunctionThis069E06354 as /var/task:ro,delegated inside runtime container
# START RequestId: fdbba255-e5b0-4e21-90d3-fe0b2ebbf629 Version: $LATEST
# sent 100 messages
# sent 100 messages
# sent 100 messages
# sent 100 messages
# sent 100 messages
# sent 100 messages
# sent 100 messages
# sent 100 messages
# sent 100 messages
# sent 100 messages
# 1000 records are sent in 11 seconds ...
# END RequestId: fdbba255-e5b0-4e21-90d3-fe0b2ebbf629
# REPORT RequestId: fdbba255-e5b0-4e21-90d3-fe0b2ebbf629  Init Duration: 0.22 ms  Duration: 12146.61 ms   Billed Duration: 12147 ms       Memory Size: 128 MB     Max Memory Used: 128 MB
# null
```

![](kpow-data-01.png#center)

![](kpow-data-02.png#center)

### Kafka App Deployment

![](kpow-overview.png#center)

![](kpow-schema.png#center)

![](eventbridge.png#center)

![](consumer.png#center)

## Summary