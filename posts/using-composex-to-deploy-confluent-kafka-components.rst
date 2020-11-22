.. title: Using ComposeX to deploy Confluent Kafka components
.. slug: using-composex-to-deploy-confluent-kafka-components
.. date: 2020-11-22 13:10:59 UTC
.. tags: AWS, ECS, Kafka, Confluent, Docker, Docker-Compose
.. category: AWS
.. link:
.. description:
.. type: text

Introduction
============

Recently started to use Apache Kafka as our messaging backbone. For those of you know me, I would probably have gone the AWS Kinesis and otherwise way,
but this was a decision I did not own.
We started using Confluent Kafka via AWS PrivateLink, which however convenient, meant we could not have some features available.

Mostly, for our developers, this meant, no Control Center and no Connect Cluster.

Confluent has done an incredible job at putting together docker images and docker-compose files which allow people to do local development and evaluations of their services.
So, as you guessed it already, it made for a perfect candidate for ECS ComposeX to take on and deploy.


The focus
=========

The focus here is on how we took the docker images published by Confluent themselves, added our own grain of salt to work with ECS and then deploy with ECS Composex.
We are going to focus mainly on Confluent Control Center (CCC) and the Connect Cluster.


Implementation
==============

.. hint::

    TLDR;
    You can find all the docker related files and examples `here <https://github.com/lambda-my-aws/composex-blog-resources/tree/confluent-apps-01/confluent-apps-01>`__


Secrets & ACLs
--------------

Least privileges is one of my most important *must-have* in technical evaluations. AWS IAM is, in my opinion, the most comprehensive system I have seen to date. So putting anything against that often makes me
perplex, as things usually do not support RBAC too well.

So, I was very happy to see that Kafka supports ACLs on topics and consumer groups, and cluster level.
I was very happy to see that the connect cluster can use 3 different service accounts to manage separately the connect cluster itself, and then use different one for the producers and for the consumers.
I was outraged to see that Control Center must have Admin level of access to the kafka cluster. There is no way to limit what people can do with it.

I was very disappointed that equally, the Confluent "Cloud" service registry does not have any notion of ACLs, but I am told by Confluent this is coming.

.. note::

    Since then, AWS has released their `AWS Glue Schema Registry <https://docs.aws.amazon.com/glue/latest/dg/schema-registry.html>`__, and will definitely get it a spin!

So, I create one service account for the connect cluster, and decided it was good enough for now to use the same service-account for all three parts (control, producer, consumer).

.. code-block::

   ccloud service-account create --description connect-cluster
   # We retrieve and keep preciously these credentials.
   ccloud api-key create --resource cluster-abcd --service-account 123456
   # Now allow some access. That is up to you, keeping it rather open for the blog copy-paste.
   for access in READ WRITE DESCRIBE; do ccloud kafka acl create --allow --service-account 123456 --operation $access --prefix --topic * ; done
   for access in READ WRITE DESCRIBE; do ccloud kafka acl create --allow --service-account 123456 --operation $access --prefix --consumer-group * ; done


.. hint::

    Refer to Confluent CLI docs for CLI usage and general ACL settings. This blog does not aim to cover best practices.
    Obviously, following least privileges access is best!

Now, we have credentials. So let's use AWS Secrets manager, given ECS integrates very well with it.
For that we have two different CFN templates which create the secret for the connect cluster and one for the control center.

.. note::

   The structure of the secrets is different from Control Center to Connect, so make sure you took note of which is which.


.. note::

   No, I do not use Vault or anything else. AWS Secrets Manager does the job these templates will eventually contain the Lambda for rotation.
   Now that AWS MSK supports SASL and Secrets Manager integration, the plan is to simulate the same here.

Now we have dealt with authorization and authentication, we can move to the easy part. Really, the above was the difficult part.


Build the Docker images
------------------------

We have the docker images build and published by Confluent, which is a somewhat difficult to follow build process, so not going to rebuild these.
But we are going to use them as source to add a few bash scripts in order to deal with the keys of the secret to be individually exposed.

These are then pushed into ECR.

.. code-block:: bash

    if [ -z ${AWS_ACCOUNT_ID+x} ]; then export AWS_ACCOUNT_ID=`(aws sts get-caller-identity | jq -r .Account)`; fi
    docker-compose build
    docker-compose push

start.sh
--------

You will have noticed a start.sh file is used to override the start of the services. The reason for it is, until recently
AWS Fargate did not support to export each individual secret JSON Key to the container.

I decided to leave it in here instead of using ComposeX to do this (which it can, see `ComposeX secrets docs. <https://docs.ecs-composex.lambda-my-aws.io/syntax/docker-compose/secrets.html>`__)
to show you how easy it is in a couple lines of bash script to export all these JSON keys at container run-time.

Typically, within your application you would import that secret and parse its JSON to get the values, however, Confluent
Images were not built for it, and fortunately, we have environment variables to override settings.

For the control-center, this also allows us to define connect clusters and therefore here, to use the one we are deploying
at the same time.

docker-compose files
--------------------

So, we have a primary docker-compose.yml file which very easily allows us to define what is constant across deployments.
Then we have an individual override file per environment. Here, just dev and stg (for staging), but then you could have 20 files if you wanted.

Now all we have to do, is deploy!

.. code-block::

   # Check the configuration is correct with the override files
   ENV_NAME=dev ecs-composex config -f docker-compose.yml -f envs/${ENV_NAME}.yml
   mkdir outputs
   # Render the CFN templates if you want to double check the content.
   ENV_NAME=dev AWS_PROFILE=myprofile ecs-composex render --format yaml -d outputs -n confluent-apps-${ENV_NAME} -f docker-compose.yml -f envs/${ENV_NAME}.yml
   # Deploy, using up
   ENV_NAME=dev AWS_PROFILE=myprofile ecs-composex up -n confluent-apps-${ENV_NAME} -f docker-compose.yml -f envs/${ENV_NAME}.yml


Here is an example of one of the envs files.

.. include:: files/confluent-apps-01/dev.yml
    :code: yaml


Conclusion
===========

Whether you are planning on using Confluent Cloud clusters or AWS MSK, thanks to their open source nature, you can deploy
Confluent components in your own AWS VPC and ECS Clusters, possibly in the future wrap them around with AppMesh if you needed,
in only a few minutes, using AWS SecretsManager to store your credentials and deploy these components, scale them in/out using ECS ComposeX.

.. hint::

    This was deployed and done with ECS ComposeX version 0.8.9 and happens to run in production today.
