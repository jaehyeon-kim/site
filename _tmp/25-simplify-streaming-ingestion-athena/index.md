<!-- Output copied to clipboard! -->

<!-----

You have some errors, warnings, or alerts. If you are using reckless mode, turn it off to see inline alerts.
* ERRORs: 0
* WARNINGs: 0
* ALERTS: 10

Conversion time: 2.671 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β34
* Mon Apr 03 2023 23:57:23 GMT-0700 (PDT)
* Source doc: Simplify Streaming Ingestion on AWS - Part 2 MSK and Athena
* Tables are currently converted to HTML tables.
* This document has images: check for >>>>>  gd2md-html alert:  inline image link in generated source and store images to your server. NOTE: Images in exported zip file from Google Docs may not appear in  the same order as they do in your doc. Please check the images!

----->


<p style="color: red; font-weight: bold">>>>>>  gd2md-html alert:  ERRORs: 0; WARNINGs: 0; ALERTS: 10.</p>
<ul style="color: red; font-weight: bold"><li>See top comment block for details on ERRORs and WARNINGs. <li>In the converted Markdown or HTML, search for inline alerts that start with >>>>>  gd2md-html alert:  for specific instances that need correction.</ul>

<p style="color: red; font-weight: bold">Links to alert messages:</p><a href="#gdcalert1">alert1</a>
<a href="#gdcalert2">alert2</a>
<a href="#gdcalert3">alert3</a>
<a href="#gdcalert4">alert4</a>
<a href="#gdcalert5">alert5</a>
<a href="#gdcalert6">alert6</a>
<a href="#gdcalert7">alert7</a>
<a href="#gdcalert8">alert8</a>
<a href="#gdcalert9">alert9</a>
<a href="#gdcalert10">alert10</a>

<p style="color: red; font-weight: bold">>>>>> PLEASE check and correct alert issues and delete this message and the inline alerts.<hr></p>



# Simplify Streaming Ingestion on AWS - Part 2 MSK and Athena

In Part 1, we discussed a streaming ingestion solution using [EventBridge](https://aws.amazon.com/eventbridge/), [Lambda](https://aws.amazon.com/lambda/), [MSK](https://aws.amazon.com/msk/) and [Redshift Serverless](https://aws.amazon.com/redshift/redshift-serverless/). Athena provides the [MSK connector](https://docs.aws.amazon.com/athena/latest/ug/connectors-msk.html) to enable SQL queries on Apache Kafka topics directly and it can also facilitate the extraction of insights without setting up an additional pipeline to store data into S3. In this post, we discuss how to update the streaming ingestion solution so that data in the Kafka topic can be queried by Athena instead of Redshift.



* [Part 1 - MSK and Redshift](https://cevo.com.au/post/streaming-ingestion-on-aws-part-1/)
* Part 2 - MSK and Athena (this post)


## Architecture

As Part 1, fake online order data is generated by multiple Lambda functions that are invoked by an EventBridge schedule rule. The schedule is set to run _every minute_ and the associating rule has a configurable number (e.g. 5) of targets. Each target points to the same Kafka producer Lambda function. In this way we are able to generate test data using multiple Lambda functions according to the desired volume of messages. Once messages are sent to a Kafka topic, they can be consumed by the Athena MSK Connector, which is a Lambda function that can be installed from the [AWS Serverless Application Repository](https://aws.amazon.com/serverless/serverlessrepo/). A new [Athena data source](https://docs.aws.amazon.com/athena/latest/ug/work-with-data-stores.html) needs to be created in order to deploy the connector and the schema of the topic should be registered with [AWS Glue Schema Registry](https://docs.aws.amazon.com/glue/latest/dg/schema-registry.html). The infrastructure is built by Terraform and the [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-terraform-support.html) is used to develop the producer Lambda function locally before deploying to AWS.



<p id="gdcalert1" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image1.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert2">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image1.png "image_tooltip")



## Infrastructure

The ingestion solution shares a large portion of infrastructure and only new resources are covered in this post. The source can be found in the [**GitHub repository**](https://github.com/jaehyeon-kim/kafka-pocs/tree/main/integration-athena) of this post.


### Glue Schema

The order data is JSON format and it has 4 attributes - _order_id_, _ordered_at_, _user_id _and _items_. Although the items attribute keeps an array of objects that includes _product_id _and _quantity_, it is specified as VARCHAR because the MSK connector doesn’t support complex types.


```
{
  "topicName": "orders",
  "message": {
    "dataFormat": "json",
    "fields": [
      {
        "name": "order_id",
        "mapping": "order_id",
        "type": "VARCHAR"
      },
      {
        "name": "ordered_at",
        "mapping": "ordered_at",
        "type": "TIMESTAMP",
        "formatHint": "yyyy-MM-dd HH:mm:ss.SSS"
      },
      {
        "name": "user_id",
        "mapping": "user_id",
        "type": "VARCHAR"
      },
      {
        "name": "items",
        "mapping": "items",
        "type": "VARCHAR"
      }
    ]
  }
}
```


The registry and schema can be created as shown below. Note the description should include the string _{AthenaFederationMSK}_ as the marker string is required for AWS Glue Registries that you use with the Amazon Athena MSK connector.


```
# integration-athena/infra/athena.tf
resource "aws_glue_registry" "msk_registry" {
  registry_name = "customer"
  description   = "{AthenaFederationMSK}"

  tags = local.tags
}

resource "aws_glue_schema" "msk_schema" {
  schema_name       = "orders"
  registry_arn      = aws_glue_registry.msk_registry.arn
  data_format       = "JSON"
  compatibility     = "NONE"
  schema_definition = jsonencode({ "topicName" : "orders", "message" : { "dataFormat" : "json", "fields" : [{ "name" : "order_id", "mapping" : "order_id", "type" : "VARCHAR" }, { "name" : "ordered_at", "mapping" : "ordered_at", "type" : "TIMESTAMP", "formatHint" : "yyyy-MM-dd HH:mm:ss.SSS" }, { "name" : "user_id", "mapping" : "user_id", "type" : "VARCHAR" }, { "name" : "items", "mapping" : "items", "type" : "VARCHAR" }] } })

  tags = local.tags
}
```



### Athena MSK Connector

In Terraform, the MSK Connector Lambda function can be created by deploying the associated CloudFormation stack from the AWS Serverless Application Repository. The stack parameters are passed into environment variables of the function and they are mostly used to establish connection to Kafka topics.


```
# integration-athena/infra/athena.tf
resource "aws_serverlessapplicationrepository_cloudformation_stack" "athena_msk_connector" {
  name             = "${local.name}-athena-msk-connector"
  application_id   = "arn:aws:serverlessrepo:us-east-1:292517598671:applications/AthenaMSKConnector"
  semantic_version = "2023.8.3"
  capabilities = [
    "CAPABILITY_IAM",
    "CAPABILITY_RESOURCE_POLICY",
  ]
  parameters = {
    AuthType           = "SASL_SSL_AWS_MSK_IAM"
    KafkaEndpoint      = aws_msk_cluster.msk_data_cluster.bootstrap_brokers_sasl_iam
    LambdaFunctionName = "${local.name}-ingest-orders"
    SpillBucket        = aws_s3_bucket.default_bucket.id
    SpillPrefix        = "athena-spill"
    SecurityGroupIds   = aws_security_group.athena_connector.id
    SubnetIds          = join(",", module.vpc.private_subnets)
    LambdaRoleARN      = aws_iam_role.athena_connector_role.arn
  }
}
```



#### Lambda Execution Role

The AWS document doesn’t include the specific IAM permissions that are necessary for the connector function and they are updated by making trials and errors. Therefore some of them are too generous and it should be refined later. 



* First it needs permission to access a MSK cluster and topics and they are copied from Part 1. 
* Next access to the Glue registry and schema is required. I consider the required permission would have been more specific if a specific registry or schema could be specified to the connector Lambda function. Rather it searches applicable registries using a string marker and that requires an additional set of permissions. 
* Then permission to the spill S3 bucket is added. I initially included a typical read/write permission on a specific bucket and objects but the Lambda function complained by throwing 403 authorized errors. Therefore I escalated the level of permissions, which is by no means acceptable in a strict environment. Further investigation is necessary for it. 
* Finally permission to get Athena query executions is added.

    ```
# integration-athena/infra/athena.tf
resource "aws_iam_role" "athena_connector_role" {
  name = "${local.name}-athena-connector-role"

  assume_role_policy = data.aws_iam_policy_document.athena_connector_assume_role_policy.json
  managed_policy_arns = [
    "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole",
    aws_iam_policy.athena_connector_permission.arn
  ]
}

data "aws_iam_policy_document" "athena_connector_assume_role_policy" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type = "Service"
      identifiers = [
        "lambda.amazonaws.com"
      ]
    }
  }
}

resource "aws_iam_policy" "athena_connector_permission" {
  name = "${local.name}-athena-connector-permission"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid = "PermissionOnCluster"
        Action = [
          "kafka-cluster:ReadData",
          "kafka-cluster:DescribeTopic",
          "kafka-cluster:Connect",
        ]
        Effect = "Allow"
        Resource = [
          "arn:aws:kafka:*:${data.aws_caller_identity.current.account_id}:cluster/*/*",
          "arn:aws:kafka:*:${data.aws_caller_identity.current.account_id}:topic/*/*"
        ]
      },
      {
        Sid = "PermissionOnGroups"
        Action = [
          "kafka:GetBootstrapBrokers"
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
      },
      {
        Sid      = "PermissionOnS3"
        Action   = ["s3:*"]
        Effect   = "Allow"
        Resource = "arn:aws:s3:::*"
      },
      {
        Sid = "PermissionOnAthenaQuery"
        Action = [
          "athena:GetQueryExecution"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}
```




#### Security Group

The security group and rules are shown below. Although the outbound rule is set to allow all protocol and port ranges, only port 443 and 9098 with the TCP protocol would be sufficient. The former is to access the Glue schema registry while the latter is for a MSK cluster with IAM authentication.


```
# integration-athena/infra/athena.tf
resource "aws_security_group" "athena_connector" {
  name   = "${local.name}-athena-connector"
  vpc_id = module.vpc.vpc_id

  lifecycle {
    create_before_destroy = true
  }

  tags = local.tags
}

resource "aws_security_group_rule" "athena_connector_msk_egress" {
  type              = "egress"
  description       = "allow outbound all"
  security_group_id = aws_security_group.athena_connector.id
  protocol          = "-1"
  from_port         = 0
  to_port           = 0
  cidr_blocks       = ["0.0.0.0/0"]
}
```



### Athena Data Source

Unfortunately connecting to MSK from Athena is yet to be supported by CloudFormation or Terraform and it is performed on AWS console as shown below. First we begin by clicking on the _Create data source_ button.



<p id="gdcalert2" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image2.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert3">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image2.png "image_tooltip")


Then we can search the Amazon MSK data source and proceed by clicking on the _Next _button.



<p id="gdcalert3" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image3.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert4">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image3.png "image_tooltip")


We can update data source details followed by selecting the connector Lambda function ARN in connection details.



<p id="gdcalert4" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image4.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert5">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image4.png "image_tooltip")


<p id="gdcalert5" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image5.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert6">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image5.png "image_tooltip")


Once the data source connection is established, we are able to see the customer database we created earlier - the Glue registry name becomes the database name.



<p id="gdcalert6" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image6.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert7">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image6.png "image_tooltip")


Also we can check the table details from the Athena editor as shown below.



<p id="gdcalert7" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image7.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert8">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image7.png "image_tooltip")



## Kafka Producer

As in Part 1, the resources related to the Kafka producer Lambda function are managed in a separate Terraform stack. This is because it is easier to build the relevant resources iteratively. Note the SAM CLI builds the whole Terraform stack even for a small change of code and it wouldn’t be convenient if the entire resources are managed in the same stack. The terraform stack of the producer is the same as Part 1 and it won’t be covered here. Only the producer Lambda function source is covered here as it is modified in order to comply with the MSK connector.


### Producer Source

The Kafka producer is created to send messages to a topic named _orders_ where fake order data is generated using the [Faker package](https://faker.readthedocs.io/en/master/). The _Order_ class generates one or more fake order records by the _create _method and an order record includes order id, order timestamp, user id and order items. Note order items are converted into string. It is because the MSK connector fails to parse them correctly. Actually the AWS document indicates the MSK connector interprets complex types as strings and I thought it would be converted into strings internally. However it turned out the list items (or array of objects) cannot be queried by Athena. Therefore it is converted into string in the first place. The Lambda function sends 100 records at a time followed by sleeping for 1 second. It repeats until it reaches MAX_RUN_SEC (e.g. 60) environment variable value. A Kafka message is made up of an order id as the key and an order record as the value. Both the key and value are serialised as JSON. Note that the stable version of the [kafka-python](https://kafka-python.readthedocs.io/en/master/index.html) package does not support the IAM authentication method. Therefore we need to install the package from a forked repository as discussed in this [GitHub issue](https://github.com/dpkp/kafka-python/pull/2255).


```
# integration-athena/kafka_producer/src/app.py
import os
import re
import datetime
import string
import json
import time
from kafka import KafkaProducer
from faker import Faker


class Order:
    def __init__(self, fake: Faker = None):
        self.fake = fake or Faker()

    def order(self):
        rand_int = self.fake.random_int(1, 1000)
        user_id = "".join(
            [string.ascii_lowercase[int(s)] if s.isdigit() else s for s in hex(rand_int)]
        )[::-1]
        return {
            "order_id": self.fake.uuid4(),
            "ordered_at": datetime.datetime.utcnow(),
            "user_id": user_id,
        }

    def items(self):
        return [
            {
                "product_id": self.fake.random_int(1, 9999),
                "quantity": self.fake.random_int(1, 10),
            }
            for _ in range(self.fake.random_int(1, 4))
        ]

    def create(self, num: int):
        return [{**self.order(), **{"items": json.dumps(self.items())}} for _ in range(num)]


class Producer:
    def __init__(self, bootstrap_servers: list, topic: str):
        self.bootstrap_servers = bootstrap_servers
        self.topic = topic
        self.producer = self.create()

    def create(self):
        return KafkaProducer(
            security_protocol="SASL_SSL",
            sasl_mechanism="AWS_MSK_IAM",
            bootstrap_servers=self.bootstrap_servers,
            value_serializer=lambda v: json.dumps(v, default=self.serialize).encode("utf-8"),
            key_serializer=lambda v: json.dumps(v, default=self.serialize).encode("utf-8"),
        )

    def send(self, orders: list):
        for order in orders:
            self.producer.send(self.topic, key={"order_id": order["order_id"]}, value=order)
        self.producer.flush()

    def serialize(self, obj):
        if isinstance(obj, datetime.datetime):
            return re.sub("T", " ", obj.isoformat(timespec="milliseconds"))
        if isinstance(obj, datetime.date):
            return str(obj)
        return obj


def lambda_function(event, context):
    if os.getenv("BOOTSTRAP_SERVERS", "") == "":
        return
    fake = Faker()
    producer = Producer(
        bootstrap_servers=os.getenv("BOOTSTRAP_SERVERS").split(","), topic=os.getenv("TOPIC_NAME")
    )
    s = datetime.datetime.now()
    ttl_rec = 0
    while True:
        orders = Order(fake).create(100)
        producer.send(orders)
        ttl_rec += len(orders)
        print(f"sent {len(orders)} messages")
        elapsed_sec = (datetime.datetime.now() - s).seconds
        if elapsed_sec > int(os.getenv("MAX_RUN_SEC", "60")):
            print(f"{ttl_rec} records are sent in {elapsed_sec} seconds ...")
            break
        time.sleep(1)
```


A sample order record is shown below.


```
{
  "order_id": "6049dc71-063b-49bd-8b68-f2326d1c8544",
  "ordered_at": "2023-03-09 21:05:00.073",
  "user_id": "febxa",
  "items": "[{\"product_id\": 4793, \"quantity\": 8}]"
}
```



## Deployment

In this section, we skip shared steps except for local development with SAM and analytics query building. See Part 1 for other steps. 


### Local Testing with SAM

To simplify development, the Eventbridge permission is disabled by setting _to_enable_trigger_ to false. Also it is shortened to loop before it gets stopped by reducing _msx_run_sec_ to 10.


```
# integration-athena/kafka_producer/variables.tf
locals {
  producer = {
    ...
    to_enable_trigger = false
    environment = {
      topic_name  = "orders"
      max_run_sec = 10
    }
  }
  ...
}
```


The Lambda function can be built with the SAM build command while specifying the hook name as terraform and enabling beta features. Once completed, it stores the build artifacts and template in the _.aws-sam_ folder.


```
$ sam build --hook-name terraform --beta-features

# Apply complete! Resources: 3 added, 0 changed, 0 destroyed.


# Build Succeeded

# Built Artifacts  : .aws-sam/build
# Built Template   : .aws-sam/build/template.yaml

# Commands you can use next
# =========================
# [*] Invoke Function: sam local invoke --hook-name terraform
# [*] Emulate local Lambda functions: sam local start-lambda --hook-name terraform
```


We can invoke the Lambda function locally using the SAM local invoke command. The Lambda function is invoked in a Docker container and the invocation logs are printed in the terminal as shown below.


```
$ sam local invoke --hook-name terraform module.kafka_producer_lambda.aws_lambda_function.this[0] --beta-features

# Experimental features are enabled for this session.
# Visit the docs page to learn more about the AWS Beta terms https://aws.amazon.com/service-terms/.

# Skipped prepare hook. Current application is already prepared.
# Invoking app.lambda_function (python3.8)
# Skip pulling image and use local one: public.ecr.aws/sam/emulation-python3.8:rapid-1.70.0-x86_64.

# Mounting .../kafka-pocs/integration-athena/kafka_producer/.aws-sam/build/ModuleKafkaProducerLambdaAwsLambdaFunctionThis069E06354 as /var/task:ro,delegated inside runtime container
# START RequestId: d800173a-ceb5-4002-be0e-6f0d9628b639 Version: $LATEST
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
# sent 100 messages
# sent 100 messages
# 1200 records are sent in 11 seconds ...
# END RequestId: d800173a-ceb5-4002-be0e-6f0d9628b639
# REPORT RequestId: d800173a-ceb5-4002-be0e-6f0d9628b639  Init Duration: 0.16 ms  Duration: 12117.64 ms   Billed Duration: 12118 ms       Memory Size: 128 MB     Max Memory Used: 128 MB
# null
```


We can also check the messages using kafka-ui.



<p id="gdcalert8" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image8.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert9">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image8.png "image_tooltip")



### Order Items Query

Below shows the query result of the orders table. The _items _column is a JSON array but it is stored as string. In order to build analytics queries, we need to flatten the array elements into rows and it is discussed below.



<p id="gdcalert9" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image9.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert10">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image9.png "image_tooltip")


We can flatten the order items using the _UNNEST _function and _CROSS JOIN_. We first need to convert it into an array type and it is implemented by parsing the column into JSON followed by type-casting it into an array in a CTE. 


```
WITH parsed AS (
    SELECT
        order_id,
        ordered_at,
        user_id,
        CAST(json_parse(items) as ARRAY(ROW(product_id INT, quantity INT))) AS items
    FROM msk.customer.orders
)
SELECT
    order_id,
    ordered_at,
    user_id,
    items_unnested.product_id,
    items_unnested.quantity
FROM parsed
CROSS JOIN unnest(parsed.items) AS t(items_unnested)
```


We can see the flattened order items as shown below.



<p id="gdcalert10" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image10.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert11">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image10.png "image_tooltip")


The remaining sections cover deploying the kafka producer Lambda, producing messages and executing an analytics query. They are skipped in this post as they are exactly and/or almost the same. See Part 1 if you would like to check it.


## Summary

Streaming ingestion to Redshift and Athena becomes much simpler thanks to new features. In this series of posts, we discussed those features by building a solution using EventBridge, Lambda, MSK, Redshift and Athena. We also covered AWS SAM integrated with Terraform for developing a Lambda function locally.