+++
author = "Jeppe Lund Andersen"
categories = ["Visual Studio Team Services", "ALM", "DevOps", "Continuous Deployment", ".NET", "Azure", "Build"]
date = 2016-07-04T06:53:29Z
description = ""
draft = true
slug = "no-downtime-continuous-deployment-setup-with-visual-studio-team-services-and-azure-part-2-3"
tags = ["Visual Studio Team Services", "ALM", "DevOps", "Continuous Deployment", ".NET", "Azure", "Build"]
title = "No Downtime, Continuous Deployment Setup With Visual Studio Team Services and Azure Part 2/3"

+++

This post is the second part out of three, on getting a complete no downtime, continuous deployment setup up and running with Visual Studio Team Services and Azure. Check out [the first part](https://nocture.dk/2016/05/21/a-no-downtime-continuous-deployment-setup-with-visual-studio-team-services-and-azure/), in case you missed it.

In this post we will set up the Continuous Integration part of the CD pipeline. In practice this means setting up VSTS to automatically test an ASP.NET solution and build a package for deployment later, if no tests fail. Looking at the overview from Part One, we are dealing with the first three steps in the pipeline:
![Continous Deployment pipeline with Visual Studio Team Services](/images/2016/10/pipelinewithservices-2.png)
The result of this post will be a CI setup based on VSTS that runs unit tests and builds a web deployment package ready for deployment. Let's get started!

### Push Some Code


### Configure Build Setup

#### Triggers

#### Tests

#### Web Deployment Package




