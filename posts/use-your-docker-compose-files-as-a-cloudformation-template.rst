.. title: Use your docker-compose files as a CloudFormation template
.. slug: use-your-docker-compose-files-as-a-cloudformation-template
.. date: 2021-01-24 17:41:21 UTC
.. tags: AWS, CloudFormation, Docker, Compose, docker-compose, Macro
.. category: AWS, CloudFormation, Macro
.. link:
.. description:
.. type: text

Introduction
===============

AWS CloudFormation features has continuously grown up over the years, more recent features such as CloudFormation modules
simply are rising the bar and providing so many more ways for Cloud Engineers to use this incredible service.

One of the features that was extremely popular (still is!) as it went out was the marriage of AWS CloudFormation and AWS Lambda
into AWS Custom resources.

At the time, so many resource types might not be supported, of options missing out of CloudFormation,
or Functions (maths or string transformations, loops etc.) which weren't native to CloudFormation abilities are now
possible.

But then, with the rise of Serverless applications, AWS Created AWS CloudFormation Macros. The most famous of which,
supported and published by AWS, is AWS::Serverless.

The idea is simple: create an object, give parameters and settings, get a Lambda function to render the appropriate resources
such as Databases, S3 buckets, subnets etc, or far simpler things, such as string functions or maths, and inject it back to
a "rendered" version of your original CloudFormation template.

Again, which AWS::Serverless, this is how one object, is, AWS::Serverless::Function, you can end up with the Function, its
IAM role, permissions set, VPC configuration etc. etc.


That, is what ECS Compose-X macro is set out to do for you, but instead of using a CloudFormation template as the source, you can
use your docker-compose files!


ECS Compose-X until now
========================

Until now, you could use ECS Compose-X as a CLI tool on your laptop or in your CICD tool of choice, get your docker-compose
files from GIT or otherwise, and transform your docker-compose file into a fully-fledge set of templates you could use
directly with AWS CloudFormation.

This is what we have been doing in my current place of work, and it has been great. But with the number of teams growing,
this means that they each have to track what the latest version of it is to fix bug or add new features support.


Introducing ECS Compose-X as a CloudFormation macro !
======================================================

With the all recent AWS Lambda support for using Docker images as the source for a Lambda function, it will allow
a number of people to very easily ship more involved and bundled up serverless applications, as well as vendors to
offer applications that can run in AWS Lambda.

.. hint::

    You should ALWAYS verify the source of a docker image before executing it. I know in 2021 we shall see some security
    breach out of developers running un-verified images in Lambda functions with administrator access....

So naturally, as I was in the process of publishing a docker image for ECS Compose-X on AWS Public ECR, I also added an image
specifically to deploy the CloudFormation macro. But, then, AWS Lambda with Images only supports images coming from a private ECR Repository...

So here are the easy to deploy links for you to install the CFN macro into your account.


.. list-table::
    :widths: 50 50
    :header-rows: 1

    * - Region
      - Lambda Layer based Macro
    * - us-east-1
      - |LAYER_US_EAST_1|
    * - eu-west-1
      - |LAYER_EU_WEST_1|

.. |DOCKER_US_EAST_1| image:: https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png
    :target: https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=compose-x-macro&templateURL=https://s3.eu-west-1.amazonaws.com/files.compose-x.io/macro/docker-macro.yaml

.. |DOCKER_EU_WEST_1| image:: https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png
    :target: https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=compose-x-macro&templateURL=https://s3.eu-west-1.amazonaws.com/files.compose-x.io/macro/docker-macro.yaml

.. |LAYER_US_EAST_1| image:: https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png
    :target: https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=compose-x-macro&templateURL=https://s3.eu-west-1.amazonaws.com/files.compose-x.io/macro/layer-macro.yaml

.. |LAYER_EU_WEST_1| image:: https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png
    :target: https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=compose-x-macro&templateURL=https://s3.eu-west-1.amazonaws.com/files.compose-x.io/macro/layer-macro.yaml

.. note::

    In the after maths of releasing this, I would recommend to go with the Layer version to allow you to perform any
    kind of audit you might want to activate any tracing that might be going on.

How to use the Macro ?
==============================

The title of this blog post is, **use your docker-compose file as a CloudFormation template**.
And this is, in essence, the objective of ECS Compose-X and the macro.

There are two ways supported to do this: using the Docker compose directly or from a remote source (S3/HTTPS).
So let's re-use our Wordpress example.


.. note::

    The following examples require for you to have installed the CFN Macro.


Using a flat file as CFN template
----------------------------------

When using your docker-compose file as a CFN template, there are a couple limitations to have in mind:

* You cannot have multiple docker-compose files together (to use override). Therefore, you would need to have a single (potentially longer) docker-compose file.
* After adding the transform section to the template, docker-compose locally will not work because **Transform** is not a valid docker-compose keyword.

As previously, I have two files:

* docker-compose.yml -- which contains our services definitions. Here, we only have our wordpress service
* aws.yml -- this is our template that sums up all the things Compose-X needs to handle for us in order to deploy the service successfully


So I am going to merge those too files together and add the following at the top (the position does not matter).


Then I add the following to the YAML file. At this point, when one gives that file to Cloudformation as a template, it will
need to run the macro to get the rendered parts of the template.

.. code-block:: yaml

    Transform:
      - compose-x

And that is all you had to do. Now we have the "template" and the Macro, we can just create a new stack (or changeset) with
AWS CloudFormation.

From the CLI

.. code-block:: bash

    CAPABILITIES="APABILITY_AUTO_EXPAND CAPABILITY_IAM CAPABILITY_NAMED_IAM"
    aws cloudformation create-stack --template-body file://merged.yml --capabilities ${CAPABILITIES} --stack-name wordpress-demo

And that's it. CloudFormation will invoke the CFN macro which will render for us all the templates we need, and return it back to
AWS CloudFormation to then create all our resources.

.. hint::

    If you have installed ECS Compose-X locally, you can merge the two files using

    .. code-block:: bash

        ecs-composex config -f docker-compose.yml -f aws.yml | tee merged.yml

.. note::

    If you are using env_files in docker-compose, you can use that in ECS Compose-X via the CLI but you cannot use it via the
    CFN macro at this time.


Using files stored in AWS S3
-----------------------------


.. code-block:: yaml

    ---
    # Wordpress demo using ComposeX Macro
    Fn::Transform:
      Name: compose-x
      Parameters:
        ComposeFiles:
          - s3://files.compose-x.io/docker-compose.yml
          - s3://files.compose-x.io/aws.yml

.. note::

    Just like with the CLI, the order in which the files are composed together (first file least priority, last highest priority)
    the order you list files in **ComposeFiles** matters in the same way.


Conclusion
===========

Given where AWS Proton is going, I feel like this is a technique that deserves more awareness from people, as anyone
today could simply write very light macros, using AWS CDK or Troposphere, or just very simple functions, and in fact do
exactly what the Proton definition is shaping up to be.
Only, doing it via AWS Lambda will allow you to solve far more complex logic than OpenAPI will ever let you.

.. note::

    Proton offers other features though. Here I am focusing only on the rendering "aspect" that both solutions have.


On the field and our day-to-day lives, what this does help with is allowing developers to have a quick glance at CloudFormation,
being able to see what the docker-compose file content is or what is the resulting version of these (stored in S3 or else).

As always, all `the source code`_ for everything is available on Github to provide you with the most visibility on what's
happening with ECS Compose-X.

In the article we will see how to the CFN macro for multi-accounts deployments and how to take advantage of it for your
CICD pipelines.

.. hint::

    Soon will be published an simple web page listing the Lambda layer versions available for you to use and the git commit
    they relate to

.. _the source code: https://github.com/lambda-my-aws/ecs_composex
