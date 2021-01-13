.. title: Docker images multi-arch manifest build with AWS CodeBuild Batch
.. slug: docker-images-multi-arch-manifest-build-with-aws-codebuild-batch
.. date: 2021-01-12 16:16:30 UTC
.. tags: Docker, AWS, CodeBuild, ARM, x86, AWS ECR
.. category:
.. link:
.. description:
.. type: text

.. contents::

Prelude
========

A few months back AWS Codebuild release Batch builds. A very easy way to build multiple things at the same time, with or without dependencies or order between each other.
Very convenient in order to avoid creating multiple projects, with different settings, simply define these settings directly in the buildspec.yml file.

Last year I read a blog post published by AWS, around docker images build for multiple architecture using AWS CodeBuild and AWS CodePipeline.   

So this is a follow-up, in some regards, to that article, to further demonstrate AWS Codebuild capabilities.


Workflow
===========

.. image:: ../../images/codebuild-batch-02/codebuild_multiarch_workflow.png
    :alt: CodeBuild batch multi-arch workflow
    :align: center



In practice
=============

It is very simple to get going with AWS CodeBuild batch, and it very well integrates into CICD Pipelines.
Here what I had to do is simply have a specific buildspec definition file to distinguish building the docker images and one for building the manifest.

Let's go step by step on how I approached the implementation


Dockerfiles
------------

At the start of the project I had only aimed to build for python3.7 images. But, I realized, why stop there.
Given that some extra commands are necessary for installing python3.8 with Amazon Linux, I thought the easiest thing for all is just to have two different files.


But in essence, they are doing the same thing: update the packages installed, install python, set the new python as default.

.. note::

   I am fully aware that this might break pre-existing packages such as **yum** but I do not intend to have additional packages installed from it.

Batch buildspec definition
--------------------------

AWS CodeBuild supports multiple configurations and is very versatile. Here we want to build 4 docker images, and gather these images in groups of two manifests.
So, we are going to have one build per configuration and therefore one image per.

Each build will end by publishing the image build to AWS ECR, which our final stage will use.

Here with a batch-graph configuration, we can define dependencies between builds. So here, our **manifest** step will only be executed once the others are finished.

.. note::

   You can define whether they all need to succeeed or not to progress.

.. code-block:: yaml

    batch:
      fast-fail: false
      build-graph:
        - identifier: amd64_py37
          env:
            compute-type: BUILD_GENERAL1_LARGE
            privileged-mode: true
            variables:
              VERSION: 3.7
              ARCH: amd64
          buildspec: build_images.yml

        - identifier: arm64v8_py37
          env:
            type: ARM_CONTAINER
            image: aws/codebuild/amazonlinux2-aarch64-standard:2.0
            compute-type: BUILD_GENERAL1_LARGE
            privileged-mode: true
            variables:
              ARCH: arm64v8
              VERSION: 3.7
          buildspec: build_images.yml

        - identifier: amd64_py38
          env:
            compute-type: BUILD_GENERAL1_LARGE
            privileged-mode: true
            variables:
              VERSION: 3.8
              ARCH: amd64
          buildspec: build_images.yml

        - identifier: arm64v8_py38
          env:
            type: ARM_CONTAINER
            image: aws/codebuild/amazonlinux2-aarch64-standard:2.0
            compute-type: BUILD_GENERAL1_LARGE
            privileged-mode: true
            variables:
              ARCH: arm64v8
              VERSION: 3.8
          buildspec: build_images.yml

        - identifier: manifest
          env:
            compute-type: BUILD_GENERAL1_LARGE
            privileged-mode: true
          depend-on:
            - amd64_py37
            - arm64v8_py37
            - amd64_py38
            - arm64v8_py38


Once the build has started, you should see

.. image:: ../../images/codebuild-batch-02/batch-outcome.jpg
    :alt: AWS Codebuild - Batch summary
    :align: center


And that is it, this is really that simple.

Possible improvements
=====================

Here I use the same image base for both python 3.7 and 3.8. So instead of doing 1 build for each, I could have simply build both images in 1 go per architecture.
But for the purpose of this example, it seemed clearer that way to demonstrate the potential of AWS Codebuild for your multi-arch and multi-os builds.


Conclusion
=============

AWS CodeBuild is growing with more and more features, and this is one that would allow a number of developers out there to very easily be able to build and publish packages
for multiple OSes and CPU architectures.

Sources
--------

You can find the source files for this project in `GitHub <https://github.com/composex/docker-python>`__
