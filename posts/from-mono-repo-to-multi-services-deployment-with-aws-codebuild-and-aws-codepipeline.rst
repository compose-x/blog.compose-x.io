.. title: From mono-repo to multi-services deployment with AWS CodeBuild and AWS Codepipeline
.. slug: from-mono-repo-to-multi-services-deployment-with-aws-codebuild-and-aws-codepipeline
.. date: 2020-11-22 17:54:27 UTC
.. tags: AWS, CodeBuild, Codepipeline, CloudFormation, ComposeX, ECS ComposeX
.. category: CICD
.. link:
.. description:
.. type: text


Introduction
============

As a follow-up to our previous blog article on CICD done at the time of the very first release of ECS ComposeX.
This time, instead of taking a hypothetical use-case (although very common) we are going to explore a use-case
I faced recently.


Developers have started a new repository and because of shared librairies or packages, custom made (and not published to AWS CodeArtifact for example)
we have a Git repository that grew with many separate folders, each containing a specific service definition.

Sometimes, one can build a unique docker-image and pass it a different command, which is very versatile, and works well for applications in Python for example.
However, other languages such as Java will require to build .jar files and such.

But then, that might be a lot of time spent building, especially if we build each image sequentially as for example, docker-compose would do.

Initial thoughts
==================


As you might know, AWS CodeBuild has recently added support for `batch builds`_: from your primary buildspec.yml file you define phases etc. as usual,
but then define multiple builds that will be triggered at the same time, in many possible combinations.

After reading the documentation and the syntax, I could however not find how, for example in a build-graph, fetch the artifact produced from a previous build
into another.

After trying a few things, using cache or otherwise, I figured that I could access each previous artifact into the next one, in the same way as you would using
AWS Codepipeline secondary inputs.


Putting it together
====================

.. hint::

   TLDR; `All files to implement the pipeline and build can be found here <https://github.com/lambda-my-aws/monorepo-blog-sources>`__

In this example, we have 2 JAVA Spring applications. I never wrote JAVA applications, I only re-used scaffolds from projects.
So these really do not do much at all, but they will serve our purpose : we will use **maven** to build our applications
JARs, which we then put into a docker image, using amazoncorretto SDK (11).

.. hint::

    These apps code is identical for the purpose of this blog post, but you could make them be whatever you want.

Workflow
---------

A good picture is worth 1000 words they say, so here to illustrate the workflow that AWS Codebuild will follow based on our
buildspec.yml file:

.. image:: ../../images/codebuild-batch-01/codebuild-workflow.jpg

`You can find the full buildspec.yml file here <https://github.com/lambda-my-aws/monorepo-blog-sources/blob/main/buildspec.yml>`__

The most import parts of the buildspec.yml is the build_identifier in the build-graph section for each.
When codebuild is done with the two previous artifacts, these will be passed onto the next phase.

In the buildspec_java_apps.yml however, we **name** the artifacts with the same name as the service, but the name of the
identifier is what matters most.

.. hint::

    It is tempting to use app_01 instead of app01 given the service name is app-01, but, from testing it around,
    this will make your life easier. Also, this allows the bash script to work to find the jars.

Builds of app01 and app02
"""""""""""""""""""""""""

As you can see in the buildspec.yml we have defined two builds which our composex phase will have to wait for completion
before moving on.

Here given we have only java applications, we can re-use the same buildspec_java_apps.yml file to build our jar files.
If we wanted to do something different for one or the other, simply create a new file for it and change the override.

ComposeX phases
""""""""""""""""""

Once app01 and app02 are complete, AWS Codebuild will kick off the **composex** phase. This does not have a buildspec.yml
override so it will use the default one, but given the batch was already evaluated at the start, it won't be evaluated
a second time around, and codebuild moves onto the **phases** of that buildspec.yml file.

Because we defined artifacts to gather in the previous stage, AWS Codebuild does something very sweet for us, but as far
as reading docs and references, is not (as I write this article) documented.

Multi-Inputs build
-------------------

For those of you used to AWS Codepipeline and AWS Codebuild, you will be familiar with how AWS Codebuild has environment
variables **CODEBUILD_SRC_{something}** which refer to the build artifacts, and more specifically, to the secondary artifacts.

Here, AWS CodeBuild very smartly simply re-used the same principle.

So if we have a build-graph with identifier **app01** we end up with **CODEBUILD_SRC_app01_AppDefinition** (AppDefinition because
it is the way it is defined in AWS Codepipeline!).

Now we know that, and given we followed a specific naming convention for our identifiers to match our services names
defined in docker-compose files, we can safely gather the outputs as needed.

Here, we only need the JAR files created by maven, and given we have a specific folder for each service source code, we
place that JAR file into the folder.

.. code-block:: bash

    for service in `docker-compose config --services 2>/dev/null`; do
          shortname=`echo $service | sed s'/[^a-zA-Z0-9]//g'`
          dir_env_name="CODEBUILD_SRC_DIR_${shortname}_AppDefinition"
          if ! [ -d ${!dir_env_name} ]; then echo "No output found for $service"; echo ${!dir_env_name}; exit 1 ; fi
          find ${!dir_env_name} -type f -name "${service}*.jar" | ( read path; cp $path ${service}/app.jar ) ;
    done

.. note::

    I chose to name the file app.jar when retrieving it from the previous build artifacts. If you modify your pom.xml
    to match your service name then that makes it even easier on you.

Bundle, publish, deploy
-----------------------

And this is where docker-compose and ComposeX really save us a lot of time and trouble.
First off, with docker-compose, we now just build the services images and push them to AWS ECR.

.. code-block:: bash

    docker-compose build
    docker-compose push

.. hint::

    Here we use the same base-image for each docker-image we build, so we do it in the composex phase to save
    time, but you could do the docker image build and publish for each service in their own "forked" build.


Once that is done, we can now use ComposeX to generate our CFN templates and configuration files. We place them into a new
artifact which the pipeline will then use.

Back to CodePipeline
---------------------

Worth pointing out, and I am yet to figure out the differences we might expect between build with batch-graph/matrix/list when it comes
to the artifacts, and how they are merged if you so wish to do so.

In this use-case, I am merging the artifacts together.

Indeed, I do not need the JAR. files in the rest of the process, but, for those of you who might want to add some lambda
functions in this repository and deploy these to layer or functions, there you go, you already have that JAR file ready in your artifacts!

CodePipeline expects cloudformation template and the config file. Given we bundled things together, CodeBuild has created
sub directories for each artifact, named based on the identifier.

We then just have to adapt our path in the CloudFormation action of the Codepipeline stage:

.. code-block:: yaml

        - Name: !Sub 'DeployToDev'
          Actions:
            - Name: CfnDeployToDev
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: '1'
              Configuration:
                ActionMode: CREATE_UPDATE
                RoleArn: !ImportValue 'CICD::nonprod::Cfn::Role::Arn'
                StackName: !Sub '${DeploymentName}-dev'
                TemplatePath: !Sub 'AppDefinition::composex/AppDefinition/dev/${DeploymentName}.yaml'
                OutputFileName: outputs.json
                TemplateConfiguration: !Sub 'AppDefinition::composex/AppDefinition/dev/${DeploymentName}.config.json'
                Capabilities: 'CAPABILITY_AUTO_EXPAND,CAPABILITY_IAM'
              InputArtifacts:
                - Name: AppDefinition
              OutputArtifacts:
                - Name: DevStackOutputs
              RunOrder: '1'
              RoleArn: !ImportValue 'CICD::nonprod::Pipeline::Role::Arn'


Conclusion
===========

We now can build multiple microservices artifacts / docker images, in parallel, and regroup the outputs of each for our
next stages in codebuild itself and later in codepipeline!

For some of the teams I work with, this is a drastic time saving and boosts efficiency as builds take way shorter amount
of time.

I hope this has been helpful in your journey to use AWS Codebuild and AWS Codepipeline, and deploy your applications via
ECS ComposeX in the mix of things!

Some thoughts before you leave
-------------------------------

* You could have a repository with your docker compose files etc. and have a repository per microservice instead of a mono repo
    and still achieve the same thing, for example, using git submodules
* If you have shared libraries you want to build first, simply add builds, publish to `AWS CodeArtifact`_ / Nexus / else
    then resume the build of your applications

.. _batch builds: https://docs.aws.amazon.com/codebuild/latest/userguide/batch-build.html
.. _AWS CodeArtifact: https://aws.amazon.com/codeartifact/
