.. title: Wordpress CMS from docker-compose to AWS in a few steps
.. slug: wordpress-cms-from-docker-compose-to-aws-in-a-few-steps
.. date: 2021-01-07 08:30:26 UTC
.. tags: AWS, CloudFormation, ECS, Wordpress, Bitnami, docker-compose, Compose, Docker
.. category: ECS ComposeX
.. link:
.. description:
.. type: text


Prelude
========

For many many years now, people have been using and modifying CMS such as Wordpress, Joomla, Magento to make it one of their own.
Over the years, they have improved dramatically to allow companies to rely on them, being so easy for anyone non-technical to publish
content on their websites.


So today we take one of the most popular(if not *the* most popular), Wordpress, and we are going to see how in a few steps we can get started
with getting it up and running in AWS ECS, using AWS RDS for database, a Load-balancer etc. to get ourselves started.


Wordpress docker images from AWS Public ECR
============================================

First off, we want to use a well known publisher of a docker image to use for our application deployment.
Since the recent release of AWS Public ECR, Bitnami, a well respected company over the years, has published their own images up there and also
provide us with the docker-compose file to get up and running.


For more information on that, head of to `Bitnami wordpress ECR page <https://gallery.ecr.aws/bitnami/wordpress>`__.


Docker-Compose file
====================

The original docker-compose file that we are provided shows us, along with the documentation on the Github pages, what this Docker image is expecting/accepting
as parameters to get the application bootstrapped.

By default, this docker-compose for local usage is going to create for us a MariaDB database to store our information.
This is setting up some data and access, which we will modify to plug into RDS.

On the application side, we are going to remove a number of environment variables which will be published to the container from the *x-rds* definition.


Adapting the docker-compose for ECS ComposeX
==============================================

When one runs docker-compose up, the environment is usually all layed out for us by docker-compose. No network management necessary (although recommended to create separate docker networks for logical placement).
Also, the database comes as a docker container itself, so very little to do and we want to keep that simplicity.


Define local environment
-------------------------

So first of all, we are going to rename our **docker-compose.yml** file **local.yml**

.. hint::

   docker-compose allows you to specify files you want to use. From the first file argument to the last specified, settings such as environment variables either merge or add up.
   This makes it very easy for us to have different configuration for different environments.


In this file we mostly want to keep the mariaDB section in order to test our changes, if any, to the main wordpress docker image.

.. code-block:: yaml

   version: '3.8'
   services:
     mariadb:
       image: 'docker.io/bitnami/mariadb:10.3-debian-10'
       volumes:
         - 'mariadb_data:/bitnami/mariadb'
       environment:
         - MARIADB_USER=bn_wordpress
         - MARIADB_DATABASE=bitnami_wordpress
         - ALLOW_EMPTY_PASSWORD=yes

     wordpress:
       depends_on:
         - mariadb
       environment:
         - MARIADB_HOST=mariadb
         - MARIADB_PORT_NUMBER=3306
         - WORDPRESS_DATABASE_USER=bn_wordpress
         - WORDPRESS_DATABASE_NAME=bitnami_wordpress
         - ALLOW_EMPTY_PASSWORD=yes


As a result in our docker-compose.yml file, we now have

.. code-block:: yaml

   version: '3.8'

   services:
     wordpress:
       image: public.ecr.aws/bitnami/wordpress:5-debian-10
       ports:
         - '8080:8080'
         - '8443:8443'
       volumes:
         - 'wordpress_data:/bitnami/wordpress'
       environment:
         WORDPRESS_SKIP_INSTALL: "yes"
       deploy:
         resources:
          reservations:
            cpus: 1.0
            memory: 1G

   volumes:
     wordpress_data:
       driver: local


.. note::

    By default, ECS ComposeX will use the smallest Fargate profile (.25 CPUs and .5G RAM). So using *deploy* as per the
    compose reference, we are assigning some more CPU and RAM to our container.

To run our wordpress locally we now would simply run

.. code-block:: bash

   # Verify the configuration of merged files
   docker-compose -f docker-compose.yml -f local.yml config
   # Deploy continers from the composed set of files
   docker-compose -f docker-compose.yml -f local.yml up


Create our AWS Environment definition
--------------------------------------

This section might seem long, but it only seems so due to a lot of explanations. You might copy-paste and adapt the code-block sections and add these to your aws.yml file.

Networking layout
++++++++++++++++++

In AWS you will need networking sorted out, using AWS VPC. If you do not already have a VPC created for you, you can let ECS ComposeX create one for you, it will do all the necessary.

In our case today, we are going to use an existing VPC. This was created before using a similar template as the one created by ComposeX, but you use your own, created manually or otherwise through IaC.
All we need is to identify our subnets we are going to deploy to.

ECS ComposeX **expects** at least three categories of subnets: **Public, Storage and Applications**.

That said, you could arrange it to place everything together. The most important thing is to make sure you
are placing your applications the most securely and, from a pure network point of view, we want to make sure that the **Public** subnets allow for traffic inbound / outbound through an Internet Gateway.

So this is how we tell ECS ComposeX how to find our VPC and Subnets:

.. code-block:: yaml

   x-vpc:
     Lookup:
       VpcId:
         Tags:
           - Name: dev-vpc
       AppSubnets:
         Tags:
           - vpc::usage: application
           - aws:cloudformation:stack-name: dev-vpc
       PublicSubnets:
         Tags:
           - vpc::usage: public
           - aws:cloudformation:stack-name: dev-vpc
       StorageSubnets:
         Tags:
           - vpc::usage: storage
           - aws:cloudformation:stack-name: dev-vpc


Now that we have that information, we write the above into a new file, calling it **aws.yml** as this refers to our AWS environment for this application.

.. hint::

   If you already know your VPC ID and Subnet IDs, you can set these via IDs using **Use** instead of **Lookup**.
   See `ECS ComposeX x-vpc syntax reference`_

.. hint::

   Refer to your network team in case you have a more complex setup.


Ingress using an Application Load-Balancer
+++++++++++++++++++++++++++++++++++++++++++++++++++++++

To anticipate for a number of things we want in order to make our site highly-available and secure, we are going to have an Application LoadBalancer.
Over time, we will update this section of the **aws.yml** file.


We start with a very basic and open configuration:

.. code-block:: yaml

   x-elbv2:
     lbA:
       Properties:
         Scheme: internet-facing
         Type: application
       Listeners:
         - Port: 80
           Protocol: HTTP
           Targets:
             - name: wordpress:wordpress
               access: /
       Services:
         - name: wordpress:wordpress
           port: 8080
           protocol: HTTP
           healthcheck: 8080:HTTP:/:7:2:15:5

.. hint::

   Refer to `ECS ComposeX x-elbv2 syntax reference`_ for a lot more details on this configuration

Database and storage
+++++++++++++++++++++

Finally, we want to use AWS RDS to store our data and for persistence of files, use AWS S3 to store our blog media content.
Now, I am no expert at WordPress, but there are a lot of ways to achieve this so, from googling around I will be using a popular
plugin that handles all that for you.

First of all, we need to define our database. AWS RDS is super powerful and we only just need a MySQL DB. At this point, you could
just use AWS RDS with MariaDB engine, AWS RDS with MySQL Engine or AWS Aurora with MySQL Engine compatibility.

Truly this is a choice to make on your own. Today to prove further compatibility, I am going to use AWS Aurora with MySQL.

.. code-block:: yaml

   x-rds:
     wordpress-db:
       Properties:
         Engine: "aurora-mysql"
         EngineVersion: "5.7"
         BackupRetentionPeriod: 1
         DatabaseName: wordpress
         StorageEncrypted: True
         Tags:
           - Key: Name
             Value: "wordpress DB"
       Services:
         - name: wordpress
           access: RW
           SecretsMappings:
             Mappings:
               host: MARIADB_HOST
               port: MARIADB_PORT_NUMBER
               username: WORDPRESS_DATABASE_USER
               password: WORDPRESS_DATABASE_PASSWORD
               dbname: WORDPRESS_DATABASE_NAME

In ECS ComposeX the best practice for passwords is, no human shall know what the password is. Only we need to use it.
Here given we use a pre-build docker image that is well documented, we know that wordpress is going to start and expect
to find some settings to connect to the databse.

Given that AWS RDS and AWS Secrets Manager marry very well, we can use well-know secret structure to expose it to the application.
That is what the **SecretsMappings** do for us here. ECS ComposeX will automatically know how to connect in AWS CloudFormation, our
secret, the keys and grant our wordpress service access to it.

.. hint::

   `ECS ComposeX x-rds syntax reference`_ for more details on that module.

Now that we have the DB sorted, let's look at the persistent storage via AWS S3. Below is a rather lenghty definition of our
S3 bucket, using the same syntax as one would already do in AWS CloudFormation templates. That is one of the keystone of ECS ComposeX: keep AWS CloudFormation compatibility.

Here, we are going to let CloudFormation decide of the bucket name for us and will get it from our outputs.

.. code-block:: yaml

   x-s3:
     wp-data-bucket:
       Properties:
         AccessControl: BucketOwnerFullControl
         ObjectLockEnabled: True
         PublicAccessBlockConfiguration:
             BlockPublicAcls: True
             BlockPublicPolicy: True
             IgnorePublicAcls: True
             RestrictPublicBuckets: False
         AccelerateConfiguration:
           AccelerationStatus: Suspended
         BucketEncryption:
           ServerSideEncryptionConfiguration:
             - ServerSideEncryptionByDefault:
                 SSEAlgorithm: AES256
       Services:
         - name: wordpress
           access:
             Bucket: ListOnly
             Objects: RW

Now at this point, you can use the following commands to merge all definitions together and see what an "all-in-one" docker-compose
definition would look like.

.. code-block:: bash

   docker-compose -f docker-compose.yml -f aws.yml config
   ecs-composex config -f docker-compose.yml -f aws.yml


.. note::

   Tried to keep the config rendering of ECS composeX as close as possible to what docker-compose would render in order to detect
   any differences in the services.
   However, docker-compose ignores all top keys starting with **x-** so you won't be able to see the rds/s3 etc. definitions.

Interlude
==========

Now let's take a small break and deploy everything as-is. Yes, it is not perfect, especially areas around security access to the
application. But, that's on purpose, in order to demonstrate how you can do some quick PoC work with ComposeX and take it up a notch
once you figured out configuration and other settings.

.. code-block:: bash

   # To make it easy for you and not configure all options, you can start by setting your AWS account up by running
   # the following command. It will set the AWS ECS settings accordingly and create a S3 bucket to store our templates.
   ecs-composex init
   # Optionally, create a folderto output your templates locally. Otherwise, they always will be a copy in /tmp
   # mkdir outputs
   # Now at the ready to deploy! (Add -d outputs to place the files in the outputs folder).
   ecs-composex up -f docker-compose.yml -f aws.yml --format yaml -n wordpress-demo

.. hint::

   If you wanted to check on the templates prior to deploying, you can use either **create** or **render** instead of **up**.
   Render will only happen locally, Create will render the files and upload them to S3.

Now take a seat back and relax, it will take a little moment for AWS to create everything for us.

After a while, all should be deployed successfully, we have an application up and running in AWS ECS from that docker image, we could see from the ECS Logs that our wordpress has started etc.

In AWS CloudWatch you should be able to find your log group for wordpress and observe logs such as

.. code-block:: bash

   2021-01-07T08:55:14.242+00:00	Welcome to the Bitnami wordpress container
   2021-01-07T08:55:14.242+00:00	Subscribe to project updates by watching https://github.com/bitnami/bitnami-docker-wordpress
   2021-01-07T08:55:14.242+00:00	Submit issues and feature requests at https://github.com/bitnami/bitnami-docker-wordpress/issues
   2021-01-07T08:55:25.033+00:00	nami INFO Initializing apache
   2021-01-07T08:55:25.237+00:00	nami INFO apache successfully initialized
   2021-01-07T08:55:36.221+00:00	nami INFO Initializing mysql-client
   2021-01-07T08:55:36.334+00:00	nami INFO mysql-client successfully initialized
   2021-01-07T08:55:50.538+00:00	nami INFO Initializing wordpress
   2021-01-07T08:55:51.018+00:00	wordpre INFO ==> Preparing Varnish environment
   2021-01-07T08:55:51.019+00:00	wordpre INFO ==> Preparing Apache environment
   2021-01-07T08:55:51.119+00:00	wordpre INFO ==> Preparing PHP environment
   2021-01-07T08:55:51.143+00:00	mysql-c INFO Trying to connect to MySQL server
   2021-01-07T08:55:51.297+00:00	mysql-c INFO Found MySQL server listening at wordpress-demo-rds-4tjz6cvkslbz-wordp-wordpressdb-174xmm2y7lwwo.cluster-cvjpaxz5wqkd.eu-west-1.rds.amazonaws.com:3306
   2021-01-07T08:55:51.319+00:00	mysql-c INFO MySQL server listening and working at wordpress-demo-rds-4tjz6cvkslbz-wordp-wordpressdb-174xmm2y7lwwo.cluster-cvjpaxz5wqkd.eu-west-1.rds.amazonaws.com:3306
   2021-01-07T08:55:51.319+00:00	wordpre INFO Preparing WordPress environment


Do more with ECS ComposeX
==========================

We know our application is up and running but currently you might be thinking that this is not particularly secure and the LB
URL is simply not user friendly.

So let's add some security and DNS configuration to point to our Wordpress.


Setup a friendly DNS Name and a SSL certificate
------------------------------------------------

.. warning::

   ECS ComposeX was built for AWS specifically so at the moment, no other DNS provider than AWS Route53 is supported.

I have a DNS domain which I already have in Route53, so I am going to simply point to it.

.. code-block:: yaml

   x-dns:
     PublicZone:
       Use: ZABCDEFGHIS0123 # Redacted for privacy purposes.
     Records:
       - Properties:
           Name: wordpress.demos.lambda-my-aws.io
           Type: A
         Target: x-elbv2::wordpress-lb



Then we can now create a new ACM Certificate

.. code-block:: yaml

   x-acm:
     wordpress-demo:
       Properties:
         DomainName: wordpress.demos.lambda-my-aws.io
         DomainValidationOptions:
           - HostedZoneId: ZABCDEFGHIS0123 # Redacted for privacy purposes
             DomainName: wordpress.demos.lambda-my-aws.io
         ValidationMethod: DNS

Now we assign this certificate to our existing ALB, simply by editing our **Listeners** section.

.. code-block:: yaml

   x-elbv2:
     lbA:
       Properties:
         Scheme: internet-facing
         Type: application
       MacroParameters:
         Ingress:
           ExtSources:
             - IPv4: 0.0.0.0/0
               Description: "ANY"
       Listeners:
         - Port: 80
           Protocol: HTTP
           DefaultActions:
             - Redirect: HTTP_TO_HTTPS
         - Port: 443
           Protocol: HTTPS
           Certificates:
             - x-acm: wordpress-demo
           Targets:
             - name: wordpress:wordpress
               access: /
       Services:
         - name: wordpress:wordpress
           port: 8080
           protocol: HTTP
           healthcheck: 8080:HTTP:/:7:2:15:5


We can simply update our existing stack to add our new ACM certificate and DNS names pointing to our ALB.

.. code-block:: bash

    ecs-composex up -f docker-compose.yml -f aws.yml --format yaml -n wordpress-demo

This update will be rather quick, and thanks to the instruction **Redirect: HTTP_TO_HTTPS** in our HTTP listener, all requests
submitted over HTTP will be redirected to HTTPs. Using an AWS ACM certificate, our clients to the Load-Balancer will be able to
get SSL information and match that against our site.

Once this is all done, setup your Wordpress user etc, install the Media Lite plugin, configure it for S3, and you can now
use the S3 bucket as storage for your media files.

Here is a little gallery to help you go through the same steps as I did to get Wordpress + S3 working.

.. gallery:: wordpress-install


Conclusion
===========

It might seem like a lot of work done to add an ALB, a certificate, a database, and point our site to the right VPC and subnets.
But, in fact, this might have taken you a few minutes to copy-paste, change some values to your liking or find out what your
DNS Public Zone ID that you can use, but once you have done that, you have nothing else to do.

ECS ComposeX does nothing magic post generating the CloudFormation templates. You can use and modify these templates manually
down the road to adapt it to your objectives.

What ECS ComposeX will do for you is handle Security Groups opening, IAM permissions, and validate a number of things, with
providing you the ability to change only very little number of things from your original docker compose file.

Here we split into multiple files only to represent multiple environments.

All the files used for this blog article can be found `here <https://github.com/compose-x/wordpress-demo>`__


What is next ?
--------------

At the time of writing this blog post, Troposphere 2.6.4 is pending release to integrate EFS for ECS. This would be
the last part of the puzzle to allow some settings to be persistent as they do not live in the database.

Also with the release announcement of AWS Proton, ECS ComposeX will focus on allowing existing Docker-compose users to
define environments by using docker-compose syntax and help with the adoption of AWS ECS to run and deploy containerized
applications.

.. _ECS ComposeX x-vpc syntax reference: https://docs.ecs-composex.lambda-my-aws.io/syntax/composex/vpc.html
.. _ECS ComposeX x-elbv2 syntax reference: https://docs.ecs-composex.lambda-my-aws.io/syntax/composex/elbv2.html
.. _ECS ComposeX x-rds syntax reference: https://docs.ecs-composex.lambda-my-aws.io/syntax/composex/rds.html
