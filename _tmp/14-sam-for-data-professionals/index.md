<!-- Output copied to clipboard! -->

<!-----

You have some errors, warnings, or alerts. If you are using reckless mode, turn it off to see inline alerts.
* ERRORs: 0
* WARNINGs: 0
* ALERTS: 4

Conversion time: 1.551 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β33
* Sun Jul 17 2022 03:17:43 GMT-0700 (PDT)
* Source doc: Serverless Application Model (SAM) for Data Professionals
* Tables are currently converted to HTML tables.
* This document has images: check for >>>>>  gd2md-html alert:  inline image link in generated source and store images to your server. NOTE: Images in exported zip file from Google Docs may not appear in  the same order as they do in your doc. Please check the images!

----->


<p style="color: red; font-weight: bold">>>>>>  gd2md-html alert:  ERRORs: 0; WARNINGs: 0; ALERTS: 4.</p>
<ul style="color: red; font-weight: bold"><li>See top comment block for details on ERRORs and WARNINGs. <li>In the converted Markdown or HTML, search for inline alerts that start with >>>>>  gd2md-html alert:  for specific instances that need correction.</ul>

<p style="color: red; font-weight: bold">Links to alert messages:</p><a href="#gdcalert1">alert1</a>
<a href="#gdcalert2">alert2</a>
<a href="#gdcalert3">alert3</a>
<a href="#gdcalert4">alert4</a>

<p style="color: red; font-weight: bold">>>>>> PLEASE check and correct alert issues and delete this message and the inline alerts.<hr></p>



# Serverless Application Model (SAM) for Data Professionals

[AWS Lambda](https://aws.amazon.com/lambda/) provides serverless computing capabilities and it can be used for performing validation or light processing/transformation of data. Moreover, with its integration with more than 140 AWS services, it facilitates building complex systems employing [event-driven architectures](https://docs.aws.amazon.com/lambda/latest/operatorguide/event-driven-architectures.html). There are many ways to build serverless applications and one of the most efficient ways is using specialised frameworks such as the [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) and [Serverless Framework](https://www.serverless.com/framework/docs). In this post, I’ll demonstrate how to build a serverless data processing application using SAM.


## Architecture

When we create an application or pipeline with AWS Lambda, most likely we’ll include its event triggers and destinations. The [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) facilitates building serverless applications by providing shorthand syntax with a number of [custom resource types](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-resources-and-properties.html). Also the AWS SAM CLI supports an execution environment that helps build, test, debug and deploy applications easily. Furthermore the CLI can be integrated with full-pledged IaaC tools such as the [AWS Cloud Development Kit (CDK)](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-cdk.html) and [Terraform](https://github.com/aws/aws-sam-cli/issues/3154) - note integration with the latter is in its roadmap. With the integration, serverless application development can be a lot easier with capabilities of local testing and building. An alternative tool is the [Serverless Framework](https://www.serverless.com/framework/docs). It supports [multiple cloud providers](https://www.serverless.com/framework/docs/providers) and broader [event sources](https://www.serverless.com/framework/docs/providers/aws/guide/events) out-of-box but its integration with IaaC tools is practically non-existent.

In this post, we’ll build a simple data pipeline using SAM where a Lambda function is triggered when an object (csv file) is created in a S3 bucket. The Lambda function converts the object into parquet and avro files and saves to a destination S3 bucket. For simplicity, we’ll use a single bucket for the source and destination.


## SAM Application

After [installing the SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html), I initialised an app with the Python 3.8 Lambda runtime from the [hello world template](https://github.com/aws/aws-sam-cli-app-templates/tree/master/python3.8/cookiecutter-aws-sam-hello-python/%7B%7Bcookiecutter.project_name%7D%7D) (`sam init --runtime python3.8`). Then it is modified for the data pipeline app. The application is defined in the _template.yaml_ and the source of the main Lambda function is placed in the _transform _folder. We need 3rd party packages for converting source files into the parquet and avro formats - [AWS Data Wrangler](https://aws-data-wrangler.readthedocs.io/en/stable/index.html) and [fastavro](https://fastavro.readthedocs.io/en/latest/). Instead of packaging them together with the Lambda function, they are made available as [Lambda layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html). While using the [AWS managed Lambda layer](https://aws-data-wrangler.readthedocs.io/en/stable/install.html#managed-layer) for the former, we only need to build the Lambda layer for the _fastavro _package and it is located in the _fastavro _folder. The source of the app can be found in the [**GitHub repository**](https://github.com/jaehyeon-kim/sam-for-data-professionals) of this post.


```
fastavro
└── requirements.txt
transform
├── __init__.py
├── app.py
└── requirements.txt
tests
├── __init__.py
└── unit
    ├── __init__.py
    └── test_handler.py
template.yaml
requirements-dev.txt
test.csv
```


In the resources section of the template, the Lambda layer for avro transformation (_FastAvro_), the main Lambda function (_TransformFunction_) and the source (and destination) S3 bucket (_SourceBucket_) are added. The layer can be built simply by adding the pip package name to the _requirements.txt_ file. It is set to be compatible with Python 3.7 to 3.9. For the Lambda function, its source is configured to be built from the _transform _folder and the ARNs of the custom and AWS managed Lambda layers are added to the layers property. Also an S3 bucket event is configured so that this Lambda function is triggered whenever a new object is created to the bucket. Finally, as it needs to have permission to read and write objects to the S3 bucket, its invocation policies are added from ready-made [policy templates](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-templates.html) - _S3ReadPolicy _and _S3WritePolicy_.


```
# template.yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  sam-for-data-professionals

  Sample SAM Template for sam-for-data-professionals

Globals:
  Function:
    MemorySize: 256
    Timeout: 20

Resources:
  FastAvro:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: fastavro-layer-py3
      ContentUri: fastavro/
      CompatibleRuntimes:
        - python3.7
        - python3.8
        - python3.9
    Metadata:
      BuildMethod: python3.8
  TransformFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: transform/
      Handler: app.lambda_handler
      Runtime: python3.8
      Layers:
        - !Ref FastAvro
        - arn:aws:lambda:ap-southeast-2:336392948345:layer:AWSDataWrangler-Python38:8
      Policies:
        - S3ReadPolicy:
            BucketName: sam-for-data-professionals-cevo
        - S3WritePolicy:
            BucketName: sam-for-data-professionals-cevo
      Events:
        BucketEvent:
          Type: S3
          Properties:
            Bucket: !Ref SourceBucket
            Events:
              - "s3:ObjectCreated:*"
  SourceBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: sam-for-data-professionals-cevo

Outputs:
  FastAvro:
    Description: "ARN of fastavro-layer-py3"
    Value: !Ref FastAvro
  TransformFunction:
    Description: "Transform Lambda Function ARN"
    Value: !GetAtt TransformFunction.Arn
```



### Lambda Function

The transform function reads an input file from the S3 bucket and saves the records as the parquet and avro formats. Thanks to the Lambda layers, we can access the necessary 3rd party packages as well as reduce the size of uploaded deployment packages and make it faster to deploy it. 


<table>
  <tr>
  </tr>
</table>



```
# transform/app.py
import re
import io
from fastavro import writer, parse_schema
import awswrangler as wr
import pandas as pd
import boto3

s3 = boto3.client("s3")

avro_schema = {
    "doc": "User details",
    "name": "User",
    "namespace": "user",
    "type": "record",
    "fields": [{"name": "name", "type": "string"}, {"name": "age", "type": "int"}],
}


def check_fields(df: pd.DataFrame, schema: dict):
    if schema.get("fields") is None:
        raise Exception("missing fields in schema keys")
    if len(set(df.columns) - set([f["name"] for f in schema["fields"]])) > 0:
        raise Exception("missing columns in schema key of fields")


def check_data_types(df: pd.DataFrame, schema: dict):
    dtypes = df.dtypes.to_dict()
    for field in schema["fields"]:
        match_type = "object" if field["type"] == "string" else field["type"]
        if re.search(match_type, str(dtypes[field["name"]])) is None:
            raise Exception(f"incorrect column type - {field['name']}")


def generate_avro_file(df: pd.DataFrame, schema: dict):
    check_fields(df, schema)
    check_data_types(df, schema)
    buffer = io.BytesIO()
    writer(buffer, parse_schema(schema), df.to_dict("records"))
    buffer.seek(0)
    return buffer


def lambda_handler(event, context):
    # get bucket and key values
    record = next(iter(event["Records"]))
    bucket = record["s3"]["bucket"]["name"]
    key = record["s3"]["object"]["key"]
    file_name = re.sub(".csv$", "", key.split("/")[-1])
    # read input csv as a data frame
    input_path = f"s3://{bucket}/{key}"
    input_df = wr.s3.read_csv([input_path])
    # write to s3 as a parquet file
    wr.s3.to_parquet(df=input_df, path=f"s3://{bucket}/output/{file_name}.parquet")
    # write to s3 as an avro file
    s3.upload_fileobj(generate_avro_file(input_df, avro_schema), bucket, f"output/{file_name}.avro")
```



### Unit Testing

We use a custom function to create avro files (_generate_avro_file_) while relying on the AWS Data Wrangler package for reading input files and writing to parquet files. Therefore unit testing is performed for the custom function only. Mainly it tests whether the avro schema matches the input data fields and data types.


```
# tests/unit/test_handler.py
import pytest
import pandas as pd
from transform import app


@pytest.fixture
def input_df():
    return pd.DataFrame.from_dict({"name": ["Vrinda", "Tracy"], "age": [22, 28]})


def test_generate_avro_file_success(input_df):
    avro_schema = {
        "doc": "User details",
        "name": "User",
        "namespace": "user",
        "type": "record",
        "fields": [{"name": "name", "type": "string"}, {"name": "age", "type": "int"}],
    }
    app.generate_avro_file(input_df, avro_schema)
    assert True


def test_generate_avro_file_fail_missing_fields(input_df):
    avro_schema = {
        "doc": "User details",
        "name": "User",
        "namespace": "user",
        "type": "record",
    }
    with pytest.raises(Exception) as e:
        app.generate_avro_file(input_df, avro_schema)
    assert "missing fields in schema keys" == str(e.value)


def test_generate_avro_file_fail_missing_columns(input_df):
    avro_schema = {
        "doc": "User details",
        "name": "User",
        "namespace": "user",
        "type": "record",
        "fields": [{"name": "name", "type": "string"}],
    }
    with pytest.raises(Exception) as e:
        app.generate_avro_file(input_df, avro_schema)
    assert "missing columns in schema key of fields" == str(e.value)


def test_generate_avro_file_fail_incorrect_age_type(input_df):
    avro_schema = {
        "doc": "User details",
        "name": "User",
        "namespace": "user",
        "type": "record",
        "fields": [{"name": "name", "type": "string"}, {"name": "age", "type": "string"}],
    }
    with pytest.raises(Exception) as e:
        app.generate_avro_file(input_df, avro_schema)
    assert f"incorrect column type - age" == str(e.value)
```




<p id="gdcalert1" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image1.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert2">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image1.png "image_tooltip")



### Build and Deploy

The app has to be built before deployment. It can be done by `sam build`.



<p id="gdcalert2" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image2.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert3">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image2.png "image_tooltip")


The deployment can be done with and without a guide. For the latter, we need to specify additional parameters such as the Cloudformation stack name, capabilities (as we create an IAM role for Lambda) and a flag to automatically determine an S3 bucket to store build artifacts. 


```
sam deploy \
  --stack-name sam-for-data-professionals \
  --capabilities CAPABILITY_IAM \
  --resolve-s3
```




<p id="gdcalert3" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image3.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert4">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image3.png "image_tooltip")




<p id="gdcalert4" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image4.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert5">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image4.png "image_tooltip")



### Trigger Lambda Function

We can simply trigger the Lambda function by uploading a source file to the S3 bucket. Once it is uploaded, we are able to see that the output parquet and avro files are saved as expected.


```
$ aws s3 cp test.csv s3://sam-for-data-professionals-cevo/input/
upload: ./test.csv to s3://sam-for-data-professionals-cevo/input/test.csv

$ aws s3 ls s3://sam-for-data-professionals-cevo/output/
2022-07-17 17:33:21        403 test.avro
2022-07-17 17:33:21       2112 test.parquet
```



## Summary

In this post, it is illustrated how to build a serverless data processing application using SAM. A Lambda function is developed, which is triggered whenever an object is created in a S3 bucket. It converts input csv files into the parquet and avro formats before saving into the destination bucket. For the format conversion, it uses 3rd party packages and they are made available by Lambda layers. The application is built and deployed and the function triggering is checked.
