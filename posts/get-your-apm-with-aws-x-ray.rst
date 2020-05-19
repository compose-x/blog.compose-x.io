.. title: Get your APM with AWS X-Ray
.. slug: get-your-apm-with-aws-x-ray
.. date: 2020-05-19 11:44:18 UTC
.. tags: ECS ComposeX, AWS CloudFormation, AWS ECS, AWS, AWS X-Ray
.. category: ECS ComposeX
.. link:
.. description:
.. type: text


New feature release: Enable AWS X-Ray for your containers
=========================================================

.. contents::
   :local:

.. hint::

   This is available since version 0.2.3

This post simply visits the new feature implemented in `ECS ComposeX`_ which allows you to turn on X-Ray for your container out of the box.

AWS X-Ray overview
------------------

`AWS X-Ray`_ is what's now one of my very favorite service on AWS. It integrates very well to pretty much any language and has some predefined integration with frameworks such as `Flask`_.

In essence, X-Ray will capture the application metrics which will enable you to identify performances issues, and also provide you with an understanding of how your services communicate
together.

It will also allow you to see how your application integrates to AWS Services.

The `AWS X-Ray`_ team also made available a Docker image that you can use in your local environments (laptops, Cloud9 etc.) and it will report metrics captured from your local environment,
so it really is flexible to integrate anywhere.


How X-Ray is added to your ECS Task
-----------------------------------

Presently, when `ECS ComposeX`_ parses the configuration and services, it will for each service create a task definition which will contain a single container definition.
Adding X-Ray was very straight forward, using the pre-defined Docker image provided by AWS, which also comes with recommened compute reservations.

When you enable X-Ray for your service in ECS ComposeX, it simply is going to add that extra container definition.

Secrets are kept secret
-----------------------

Because I care about security, and I am sure you do too, in the code is implemented to ensure that the X-Ray container will not be exposed with Secrets.
For example, if you service was linked to a RDS DB, which would expose the secret as an environment variable to the container, the X-Ray container is specifically identified
to not have access to that secret too.

IAM policy
----------

The IAM policy that allows the X-Ray container / app to communicate with the X-Ray service is added to the IAM Task Role.


Enable X-Ray for your service
-----------------------------

Enable or disable locally for a specific service

.. code-block:: yaml

   services:
     serviceA:
       image: link_to_image
       configs:
         x-ray:
	   enabled: true




Enable for all services from the top level:

.. code-block:: yaml

   services:
     serviceA:
       image: link_to_image

   configs:
     composex:
       x-ray:
         enabled: true

And yes, it is as simple as that.

What is next ?
==============

Currently working on implement some more fundamentals features coming from the Docker compose definition and implementing helpers that will simplify Scaling defintions of the services.

Your feedback is most welcome and this project features will be prioritized based on what's needed from its users.

.. _ECS ComposeX: https://pypi.org/project/ecs-composex/
.. _AWS X-Ray: https://aws.amazon.com/xray/
.. _Flask: https://pypi.org/project/Flask/
