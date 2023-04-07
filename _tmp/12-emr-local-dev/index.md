<!-- Output copied to clipboard! -->

<!-----

You have some errors, warnings, or alerts. If you are using reckless mode, turn it off to see inline alerts.
* ERRORs: 0
* WARNINGs: 0
* ALERTS: 9

Conversion time: 4.337 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β33
* Sun Jul 17 2022 03:28:17 GMT-0700 (PDT)
* Source doc: Develop and Test Apache Spark Apps for EMR Locally Using Docker
* Tables are currently converted to HTML tables.
* This document has images: check for >>>>>  gd2md-html alert:  inline image link in generated source and store images to your server. NOTE: Images in exported zip file from Google Docs may not appear in  the same order as they do in your doc. Please check the images!


WARNING:
You have 4 H1 headings. You may want to use the "H1 -> H2" option to demote all headings by one level.

----->


<p style="color: red; font-weight: bold">>>>>>  gd2md-html alert:  ERRORs: 0; WARNINGs: 1; ALERTS: 9.</p>
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

<p style="color: red; font-weight: bold">>>>>> PLEASE check and correct alert issues and delete this message and the inline alerts.<hr></p>



# Develop and Test Apache Spark Apps for EMR Locally Using Docker

[Amazon EMR](https://aws.amazon.com/emr/) is a managed service that simplifies running Apache Spark on AWS. It has multiple deployment options that cover EC2, [EKS](https://aws.amazon.com/emr/features/eks/), [Outposts](https://aws.amazon.com/emr/features/outposts/) and [Serverless](https://aws.amazon.com/emr/serverless/). For development and testing, [EMR Notebooks](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-managed-notebooks.html) or [EMR Studio](https://aws.amazon.com/emr/features/studio/) can be an option. Both provide a Jupyter Notebook environment and the former is only available for EMR on EC2. There are cases, however, that development (and learning) is performed in a local environment more efficiently. The AWS Glue team understands this demand and they illustrate how to make use of a custom Docker image for Glue in a [recent blog post](https://aws.amazon.com/blogs/big-data/develop-and-test-aws-glue-version-3-0-jobs-locally-using-a-docker-container/). However we don’t hear similar news from the EMR team. In order to fill the gap, we’ll discuss how to create a Spark local development environment for EMR using Docker and/or VSCode. Typical Spark development examples will be demonstrated, which covers Spark Submit, pytest, PySpark shell, Jupyter Notebook and Spark Structured Streaming. For the Spark Submit and Jupyter Notebook examples, [Glue Catalog integration](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-spark-glue.html) will be illustrated as well. And both the cases of utilising [Visual Studio Code Remote - Containers](https://code.visualstudio.com/docs/remote/containers) extension and running as an isolated container will be covered in some key examples.


# Custom Docker Image

While we may build a custom Spark Docker image from scratch, it’ll be tricky to configure the [AWS Glue Data Catalog as the metastore for Spark SQL](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-spark-glue.html). Note that it is important to set up this feature because it can be used to integrate other AWS services such as Athena, Glue, Redshift Spectrum and so on. For example, with this feature, we can create a Glue table using a Spark application and the table can be queried by Athena or Redshift Spectrum. 

Instead we can use one of the [Docker images for EMR on EKS](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/emr-eks-releases.html) as a base image and build a custom image from it. As indicated in the [EMR on EKS document](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/docker-custom-images.html), we can pull an EMR release image from ECR. Note to select the right AWS account ID as it is different from one region to another. After authenticating to the ECR repository, I pulled the latest EMR 6.5.0 release image.


```
## different aws region has a different account id 
$ aws ecr get-login-password --region ap-southeast-2 \
  | docker login --username AWS --password-stdin 038297999601.dkr.ecr.ap-southeast-2.amazonaws.com
## download the latest release (6.5.0)
$ docker pull 038297999601.dkr.ecr.ap-southeast-2.amazonaws.com/spark/emr-6.5.0:20211119
```


In the [Dockerfile](https://github.com/jaehyeon-kim/emr-local-dev/blob/main/.devcontainer/Dockerfile), I updated the default user (_hadoop_) to have the admin privilege as it can be handy to modify system configuration if necessary. Then [spark-defaults.conf](https://github.com/jaehyeon-kim/emr-local-dev/blob/main/.devcontainer/spark/spark-defaults.conf) and [log4j.properties](https://github.com/jaehyeon-kim/emr-local-dev/blob/main/.devcontainer/spark/log4j.properties) are copied to the Spark configuration folder - they’ll be discussed in detail below. Finally a number of python packages are installed. Among those, the [ipykernel](https://pypi.org/project/ipykernel/) and [python-dotenv](https://pypi.org/project/python-dotenv/) packages are installed to work on Jupyter Notebooks and the [pytest](https://pypi.org/project/pytest/) and [pytest-cov](https://pypi.org/project/pytest-cov/) packages are for testing. The custom Docker image is built with the following command: `docker build -t=emr-6.5.0:20211119 .devcontainer/`.


```
# .devcontainer/Dockerfile
FROM 038297999601.dkr.ecr.ap-southeast-2.amazonaws.com/spark/emr-6.5.0:20211119

USER root

## Add hadoop to sudo
RUN yum install -y sudo git \
  && echo "hadoop ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

## Update spark config and log4j properties
COPY ./spark/spark-defaults.conf /usr/lib/spark/conf/spark-defaults.conf
COPY ./spark/log4j.properties /usr/lib/spark/conf/log4j.properties

## Install python packages
COPY ./pkgs /tmp/pkgs
RUN pip3 install -r /tmp/pkgs/requirements.txt

USER hadoop:hadoop
```


In the default spark configuration file (_spark-defaults.conf_) shown below, I commented out the following properties that are strictly related to EMR on EKS.



* _spark.master_
* _spark.submit.deployMode_
* _spark.kubernetes.container.image.pullPolicy_
* _spark.kubernetes.pyspark.pythonVersion _

Then I changed the custom AWS credentials provider class from [WebIdentityTokenCredentialsProvider](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/WebIdentityTokenCredentialsProvider.html) to [EnvironmentVariableCredentialsProvider](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/EnvironmentVariableCredentialsProvider.html). Note EMR jobs are run by a service account on EKS and authentication is managed by web identity token credentials. In a local environment, however, we don’t have an identity provider to authenticate so that access via environment variables can be an easy alternative option. We need the following environment variables to access AWS resources.



* _AWS_ACCESS_KEY_ID_
* _AWS_SECRET_ACCESS_KEY_
* _AWS_SESSION_TOKEN_
    * note it is optional and required if authentication is made via assume role
* _AWS_REGION_
    * note it is NOT _AWS_DEFAULT_REGION_

Finally I enabled Hive support and set _AWSGlueDataCatalogHiveClientFactory_ as the Hive metastore factory class. When we start an EMR job, we can [override application configuration](https://docs.aws.amazon.com/emr-on-eks/latest/APIReference/API_ConfigurationOverrides.html) to use AWS Glue Data Catalog as the metastore for Spark SQL and these are the relevant configuration changes for it.


```
# .devcontainer/spark/spark-defaults.conf

...

#spark.master                     k8s://https://kubernetes.default.svc:443
#spark.submit.deployMode          cluster
spark.hadoop.fs.defaultFS        file:///
spark.shuffle.service.enabled    false
spark.dynamicAllocation.enabled  false
#spark.kubernetes.container.image.pullPolicy  Always
#spark.kubernetes.pyspark.pythonVersion 3
spark.hadoop.fs.s3.customAWSCredentialsProvider  com.amazonaws.auth.EnvironmentVariableCredentialsProvider
spark.hadoop.dynamodb.customAWSCredentialsProvider  com.amazonaws.auth.EnvironmentVariableCredentialsProvider
spark.authenticate               true
## for Glue catalog
spark.sql.catalogImplementation  hive
spark.hadoop.hive.metastore.client.factory.class  com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory
```


Even if the credentials provider class is changed, it keeps showing long warning messages while fetching EC2 metadata. The following lines are added to the Log4j properties in order to disable those messages.


```
# .devcontainer/spark/log4j.properties

...

## Ignore warn messages related to EC2 metadata access failure
log4j.logger.com.amazonaws.internal.InstanceMetadataServiceResourceFetcher=FATAL
log4j.logger.com.amazonaws.util.EC2MetadataUtils=FATAL
```



# VSCode Development Container

We are able to run Spark Submit, pytest, PySpark shell examples as an isolated container using the custom Docker image. However it can be much more convenient if we are able to perform development inside the Docker container where our app is executed. The [Visual Studio Code Remote - Containers](https://code.visualstudio.com/docs/remote/containers) extension allows you to open a folder inside a container and to use VSCode’s feature sets. It supports both a standalone container and [Docker Compose](https://code.visualstudio.com/docs/remote/create-dev-container#_use-docker-compose). In this post, we’ll use the latter as we’ll discuss an example Spark Structured Streaming application and multiple services should run and linked together for it.


## Docker Compose

The main service (container) is named _spark_ and its command prevents it from being terminated. The current working directory is mapped to _/home/hadoop/repo_ and it’ll be the container folder that we’ll open for development. The aws configuration folder is volume-mapped to the container user’s home directory. It is an optional configuration to access AWS services without relying on AWS credentials via environment variables. The remaining services are related to Kafka. The _kafka_ and _zookeeper_ services are to run a Kafka cluster and the _kafka-ui_ allows us to access the cluster on a browser. The services share the same Docker network named _spark_. Note that the compose file includes other Kafka related services and their details can be found in [one of my earlier posts](https://cevo.com.au/post/external-schema-registry-part-1/).


```
# .devcontainer/docker-compose.yml
version: "2"

services:
  spark:
    image: emr-6.5.0:20211119
    container_name: spark
    command: /bin/bash -c "while sleep 1000; do :; done"
    networks:
      - spark
    volumes:
      - ${PWD}:/home/hadoop/repo
      - ${HOME}/.aws:/home/hadoop/.aws
  zookeeper:
    image: bitnami/zookeeper:3.7.0
    container_name: zookeeper
    ports:
      - "2181:2181"
    networks:
      - spark
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
  kafka:
    image: bitnami/kafka:2.8.1
    container_name: kafka
    ports:
      - "9092:9092"
    networks:
      - spark
    environment:
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper
  kafka-ui:
    image: provectuslabs/kafka-ui:0.3.3
    container_name: kafka-ui
    ports:
      - "8080:8080"
    networks:
      - spark
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
      ...
    depends_on:
      - zookeeper
      - kafka
...

networks:
  spark:
    name: spark
```



## Development Container

The development container is configured to connect the _spark _service among the Docker Compose services. The _AWS_PROFILE _environment variable is optionally set for AWS configuration and additional folders are added to _PYTHONPATH_, which is to use the bundled pyspark and py4j packages of the Spark distribution. The port 4040 for Spark History Server is added to the forwarded ports array - I guess it’s optional as the port is made accessible in the compose file. The remaining sections are for installing VSCode extensions and adding editor configuration. Note we need the Python extension (_ms-python.python_) not only for code formatting but also for working on Jupyter Notebooks.


```
# .devcontainer/devcontainer.json
{
  "name": "Spark Development",
  "dockerComposeFile": "docker-compose.yml",
  "service": "spark",
  "runServices": [
    "spark",
    "zookeeper",
    "kafka",
    "kafka-ui"
  ],
  "remoteEnv": {
    "AWS_PROFILE": "cevo",
    "PYTHONPATH": "/usr/lib/spark/python/lib/py4j-0.10.9-src.zip:/usr/lib/spark/python/"
  },
  "workspaceFolder": "/home/hadoop/repo",
  "extensions": ["ms-python.python", "esbenp.prettier-vscode"],
  "forwardPorts": [4040],
  "settings": {
    "terminal.integrated.profiles.linux": {
      "bash": {
        "path": "/bin/bash"
      }
    },
    "terminal.integrated.defaultProfile.linux": "bash",
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.tabSize": 2,
    "python.defaultInterpreterPath": "python3",
    "python.testing.pytestEnabled": true,
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": false,
    "python.linting.flake8Enabled": false,
    "python.formatting.provider": "black",
    "python.formatting.blackPath": "black",
    "python.formatting.blackArgs": ["--line-length", "100"],
    "[python]": {
      "editor.tabSize": 4,
      "editor.defaultFormatter": "ms-python.python"
    }
  }
}
```


We can open the current folder in the development container after launching the Docker Compose services by executing the following command in the [command palette](https://code.visualstudio.com/docs/getstarted/userinterface#_command-palette).



* _Remote-Containers: Open Folder in Container..._



<p id="gdcalert1" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image1.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert2">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image1.png "image_tooltip")


Once the development container is ready, the current folder will be open within the spark service container. We are able to check the container’s current folder is _/home/hadoop/repo_ and the container user is _hadoop_.



<p id="gdcalert2" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image2.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert3">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image2.png "image_tooltip")



## File Permission Management

I use Ubuntu in WSL 2 for development and the user ID and group ID of my WSL user are 1000. On the other hand, the container user is _hadoop_ and its user ID and group ID are 999 and 1000 respectively. When you create a file in the host, the user has the read and write permissions of the file while the group only has the read permission. Therefore you can read the file inside the development container by the container user but it is not possible to modify it due to lack of the write permission. This file permission issue will happen when a file is created by the container user and the WSL user tries to modify it in the host. A quick search shows this is a typical behaviour applicable only to Linux (not Mac or Windows). 

In order to handle this file permission issue, we can update the file permission so that the read and write permissions are given to both the user and group. Note the host (WSL) user and container user have the same group ID and writing activities will be allowed at least by the group permission. Below shows an example. The read and write permissions for files in the project folder are given to both the user and group. Those that are created by the container user indicate the user name while there are 2 files that are created by the WSL user and it is indicated by the user ID because there is no user whose user ID is 1000 in the container.


```
bash-4.2$ ls -al | grep '^-'
-rw-rw-r--  1 hadoop hadoop 1086 Apr 12 22:23 .env
-rw-rw-r--  1   1000 hadoop 1855 Apr 12 19:45 .gitignore
-rw-rw-r--  1   1000 hadoop   66 Mar 30 22:39 README.md
-rw-rw-r--  1 hadoop hadoop  874 Apr  5 11:14 test_utils.py
-rw-rw-r--  1 hadoop hadoop 3882 Apr 12 22:24 tripdata.ipynb
-rw-rw-r--  1 hadoop hadoop 1653 Apr 24 13:09 tripdata_notify.py
-rw-rw-r--  1 hadoop hadoop 1101 Apr 24 01:22 tripdata.py
-rw-rw-r--  1 hadoop hadoop  664 Apr 12 19:45 utils.py
```


Below is the same file list that is printed in the host. Note that the group name is changed into the WSL user’s group and those that are created by the container user are marked by the user ID.


```
jaehyeon@cevo:~/personal/emr-local-dev$ ls -al | grep '^-'
-rw-rw-r--  1      999 jaehyeon 1086 Apr 13 08:23 .env
-rw-rw-r--  1 jaehyeon jaehyeon 1855 Apr 13 05:45 .gitignore
-rw-rw-r--  1 jaehyeon jaehyeon   66 Mar 31 09:39 README.md
-rw-rw-r--  1      999 jaehyeon  874 Apr  5 21:14 test_utils.py
-rw-rw-r--  1      999 jaehyeon 3882 Apr 13 08:24 tripdata.ipynb
-rw-rw-r--  1      999 jaehyeon 1101 Apr 24 11:22 tripdata.py
-rw-rw-r--  1      999 jaehyeon 1653 Apr 24 23:09 tripdata_notify.py
-rw-rw-r--  1      999 jaehyeon  664 Apr 13 05:45 utils.py
```


We can add the read or write permission of a single file or a folder easily as shown below - `g+rw`. Note the last example is for the AWS configuration folder and only the read access is given to the group. Note also that file permission change is not affected if the repository is cloned into a new place and thus it only affects the local development environment.


```
# add write access of a file to the group
sudo chmod g+rw /home/hadoop/repo/<file-name>
# add write access of a folder to the group
sudo chmod -R g+rw /home/hadoop/repo/<folder-name>
# add read access of the .aws folder to the group
sudo chmod -R g+r /home/hadoop/.aws
```



# Examples

In this section, I’ll demonstrate typical Spark development examples. They’ll cover Spark Submit, pytest, PySpark shell, Jupyter Notebook and Spark Structured Streaming. For the Spark Submit and Jupyter Notebook examples, Glue Catalog integration will be illustrated as well. And both the cases of utilising Visual Studio Code Remote - Containers extension and running as an isolated container will be covered in some key examples.


## Spark Submit

It is a simple Spark application that reads a sample NY taxi trip dataset from a public S3 bucket. Once loaded, it converts the pick-up and drop-off datetime columns from string to timestamp followed by writing the transformed data to a destination S3 bucket. The destination bucket name (_bucket_name_) can be specified by a system argument or its default value is taken. It finishes by creating a Glue table and, similar to the destination bucket name, the table name (_tblname_) can be specified as well.


```
# tripdata.py
import sys
from pyspark.sql import SparkSession

from utils import to_timestamp_df

if __name__ == "__main__":
    spark = SparkSession.builder.appName("Trip Data").getOrCreate()

    dbname = "tripdata"
    tblname = "ny_taxi" if len(sys.argv) <= 1 else sys.argv[1]
    bucket_name = "emr-local-dev" if len(sys.argv) <= 2 else sys.argv[2]
    dest_path = f"s3://{bucket_name}/{tblname}/"
    src_path = "s3://aws-data-analytics-workshops/shared_datasets/tripdata/"
    # read csv
    ny_taxi = spark.read.option("inferSchema", "true").option("header", "true").csv(src_path)
    ny_taxi = to_timestamp_df(ny_taxi, ["lpep_pickup_datetime", "lpep_dropoff_datetime"])
    ny_taxi.printSchema()
    # write parquet
    ny_taxi.write.mode("overwrite").parquet(dest_path)
    # create glue table
    ny_taxi.registerTempTable(tblname)
    spark.sql(f"CREATE DATABASE IF NOT EXISTS {dbname}")
    spark.sql(f"USE {dbname}")
    spark.sql(
        f"""CREATE TABLE IF NOT EXISTS {tblname}
            USING PARQUET
            LOCATION '{dest_path}'
            AS SELECT * FROM {tblname}
        """
    )
```


The Spark application can be submitted as shown below. 


```
export AWS_ACCESS_KEY_ID=<AWS-ACCESS-KEY-ID>
export AWS_SECRET_ACCESS_KEY=<AWS-SECRET-ACCESS-KEY>
export AWS_REGION=<AWS-REGION>
# optional
export AWS_SESSION_TOKEN=<AWS-SESSION-TOKEN>

$SPARK_HOME/bin/spark-submit \
  --deploy-mode client \
  --master local[*] \
  tripdata.py
```


Once it completes, the Glue table will be created and we can query it using Athena as shown below.



<p id="gdcalert3" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image3.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert4">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image3.png "image_tooltip")


If we want to submit the application as an isolated container, we can use the custom image directly. Below shows the equivalent Docker run command.


```
docker run --rm \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  -e AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN \ # optional
  -e AWS_REGION=$AWS_REGION \
  -v $PWD:/usr/hadoop \
  emr-6.5.0:20211119 \
  /usr/lib/spark/bin/spark-submit --deploy-mode client --master local[*] /usr/hadoop/tripdata.py taxi emr-local-dev
```



## Pytest

The Spark application in the earlier example uses a custom function that converts the data type of one or more columns from string to timestamp - `to_timestamp_df()`. The source of the function and the testing script of it can be found below.


```
# utils.py
from typing import List, Union
from pyspark.sql import DataFrame
from pyspark.sql.functions import col, to_timestamp

def to_timestamp_df(
    df: DataFrame, fields: Union[List[str], str], format: str = "M/d/yy H:mm"
) -> DataFrame:
    fields = [fields] if isinstance(fields, str) else fields
    for field in fields:
        df = df.withColumn(field, to_timestamp(col(field), format))
    return df
# test_utils.py
import pytest
import datetime
from pyspark.sql import SparkSession
from py4j.protocol import Py4JError

from utils import to_timestamp_df

@pytest.fixture(scope="session")
def spark():
    return (
        SparkSession.builder.master("local")
        .appName("test")
        .config("spark.submit.deployMode", "client")
        .getOrCreate()
    )


def test_to_timestamp_success(spark):
    raw_df = spark.createDataFrame(
        [("1/1/17 0:01",)],
        ["date"],
    )

    test_df = to_timestamp_df(raw_df, "date", "M/d/yy H:mm")
    for row in test_df.collect():
        assert row["date"] == datetime.datetime(2017, 1, 1, 0, 1)


def test_to_timestamp_bad_format(spark):
    raw_df = spark.createDataFrame(
        [("1/1/17 0:01",)],
        ["date"],
    )

    with pytest.raises(Py4JError):
        to_timestamp_df(raw_df, "date", "M/d/yy HH:mm").collect()
```


As the test cases don’t access AWS services, they can be executed simply by the pytest command (eg `pytest -v`).  



<p id="gdcalert4" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image4.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert5">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image4.png "image_tooltip")


Testing can also be made in an isolated container as shown below. Note that we need to add the _PYTHONPATH _environment variable because we use the bundled pyspark package.


```
docker run --rm \
  -e PYTHONPATH="/usr/lib/spark/python/lib/py4j-0.10.9-src.zip:/usr/lib/spark/python/" \
  -v $PWD:/usr/hadoop \
  emr-6.5.0:20211119 \
  pytest /usr/hadoop -v
```



## PySpark Shell

The PySpark shell can be launched as shown below. 


```
$SPARK_HOME/bin/pyspark \
  --deploy-mode client \
  --master local[*]
```




<p id="gdcalert5" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image5.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert6">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image5.png "image_tooltip")


Also below shows an example of launching it as an isolated container.


```
docker run --rm -it \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  -e AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN \ # optional
  -e AWS_REGION=$AWS_REGION \
  -v $PWD:/usr/hadoop \
  emr-6.5.0:20211119 \
  /usr/lib/spark/bin/pyspark --deploy-mode client --master local[*]
```



## Jupyter Notebook

Jupyter Notebook is a popular Spark application authoring tool and we can create a notebook simply by [creating a file with the ipynb extension](https://code.visualstudio.com/docs/datascience/jupyter-notebooks#_create-or-open-a-jupyter-notebook) in VSCode. Note we need the _ipykernel_ package in order to run code cells and it is already installed in the custom Docker image. For accessing AWS resources, we need the environment variables of AWS credentials mentioned earlier. We can use the _python-dotenv_ package. Specifically we can create an _.env_ file and add AWS credentials to it. Then we can add a [code cell that loads the .env file](https://github.com/theskumar/python-dotenv#load-env-files-in-ipython) at the beginning of the notebook.

In the next code cell, the app reads the Glue table and adds a column of trip duration followed by showing the summary statistics of key columns. We see some puzzling records that show zero trip duration or negative total amount. Among those, we find negative total amount records should be reported immediately and a Spark Structured Streaming application turns out to be a good option.



<p id="gdcalert6" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image6.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert7">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image6.png "image_tooltip")




<p id="gdcalert7" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image7.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert8">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image7.png "image_tooltip")



## Spark Streaming

We need sample data that can be read by the Spark application. In order to generate it, the individual records are taken from the source CSV file and saved locally after being converted into json. Below script creates those json files in the _data/json_ folder. Inside the development container, it can be executed as `python3 data/generate.py`.


```
# data/generate.py
import shutil
import io
import json
import csv
from pathlib import Path
import boto3

BUCKET_NAME = "aws-data-analytics-workshops"
KEY_NAME = "shared_datasets/tripdata/tripdata.csv"
DATA_PATH = Path.joinpath(Path(__file__).parent, "json")


def recreate_data_path_if(data_path: Path, recreate: bool = True):
    if recreate:
        shutil.rmtree(data_path, ignore_errors=True)
        data_path.mkdir()


def write_to_json(bucket_name: str, key_name: str, data_path: Path, recreate: bool = True):
    s3 = boto3.resource("s3")
    data = io.BytesIO()
    bucket = s3.Bucket(bucket_name)
    bucket.download_fileobj(key_name, data)
    contents = data.getvalue().decode("utf-8")
    print("download complete")
    reader = csv.DictReader(contents.split("\n"))
    recreate_data_path_if(data_path, recreate)
    for c, row in enumerate(reader):
        record_id = str(c).zfill(5)
        data_path.joinpath(f"{record_id}.json").write_text(
            json.dumps({**{"record_id": record_id}, **row})
        )


if __name__ == "__main__":
    write_to_json(BUCKET_NAME, KEY_NAME, DATA_PATH, True)
```


In the Spark streaming application, the steam reader loads json files in the _data/json_ folder and the data schema is provided by DDL statements. Then it generates the target dataframe that filters records whose total amount is negative. Note the target dataframe is structured to have the key and value columns, which is required by Kafka. Finally it writes the records of the target dataframe to the _notifications _topics of the Kafka cluster. 


```
# tripdata_notify.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, to_json, struct
from utils import remove_checkpoint

if __name__ == "__main__":
    remove_checkpoint()

    spark = (
        SparkSession.builder.appName("Trip Data Notification")
        .config("spark.streaming.stopGracefullyOnShutdown", "true")
        .config("spark.sql.shuffle.partitions", 3)
        .getOrCreate()
    )

    tripdata_ddl = """
    record_id STRING,
    VendorID STRING,
    lpep_pickup_datetime STRING,
    lpep_dropoff_datetime STRING,
    store_and_fwd_flag STRING,
    RatecodeID STRING,
    PULocationID STRING,
    DOLocationID STRING,
    passenger_count STRING,
    trip_distance STRING,
    fare_amount STRING,
    extra STRING,
    mta_tax STRING,
    tip_amount STRING,
    tolls_amount STRING,
    ehail_fee STRING,
    improvement_surcharge STRING,
    total_amount STRING,
    payment_type STRING,
    trip_type STRING
    """

    ny_taxi = (
        spark.readStream.format("json")
        .option("path", "data/json")
        .option("maxFilesPerTrigger", "1000")
        .schema(tripdata_ddl)
        .load()
    )

    target_df = ny_taxi.filter(col("total_amount") <= 0).select(
        col("record_id").alias("key"), to_json(struct("*")).alias("value")
    )

    notification_writer_query = (
        target_df.writeStream.format("kafka")
        .queryName("notifications")
        .option("kafka.bootstrap.servers", "kafka:9092")
        .option("topic", "notifications")
        .outputMode("append")
        .option("checkpointLocation", ".checkpoint")
        .start()
    )

    notification_writer_query.awaitTermination()
```


The streaming application can be submitted as shown below. Note the [Kafak 0.10+ Source for Structured Streaming](https://mvnrepository.com/artifact/org.apache.spark/spark-sql-kafka-0-10_2.12/3.1.2) and its dependencies are added directly to the spark submit command as indicated by the [official document](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html#deploying).  


```
$SPARK_HOME/bin/spark-submit \
  --deploy-mode client \
  --master local[*] \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.1.2 \
  tripdata_notify.py
```


We can check the topic via Kafka UI on port 8080. We see the notifications topic has 50 messages, which matches to the number that we obtained from the notebook. 



<p id="gdcalert8" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image8.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert9">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image8.png "image_tooltip")


We can check the individual messages via the UI as well.



<p id="gdcalert9" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image9.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert10">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image9.png "image_tooltip")



# Summary

In this post, we discussed how to create a Spark local development environment for EMR using Docker and/or VSCode. A range of Spark development examples are demonstrated and Glue Catalog integration is illustrated in some of them. And both the cases of utilising Visual Studio Code Remote - Containers extension and running as an isolated container are covered in some key examples.