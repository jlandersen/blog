---
title: "Deploying ASP.NET Core on AWS Fargate with Github Actions"
slug: "deploying-aspnetcore-aws-fargate-with-github-actions"
date: 2019-09-27T20:52:38+02:00
draft: true
tags: [".NET Core", "ASP.NET Core", "DevOps", "AWS", "Fargate", "continuous deployment"]
---

GitHub Actions is the new kid on the block for CI/CD. In this post we will see how easy it is to set up a continuous deployment pipeline, continuously deploying an ASP.NET Core application to AWS Fargate when code is pushed. The goal is to show how far you can get with very little, for even the smallest projects. We will use nothing more than GitHub, AWS ECR for hosting the container images and ECS for serving the app. This is the workflow we will build:
![](/images/2019/11/pipeline.png)

## Setup: Create ECR and ECS Resources
Before creating the workflow, let's set up the necessary AWS resources. We'll do this using Terraform, so this post assumes some familiarity with this. If you prefer CDK, CloudFormation, regular scripting with the AWS CLI or maybe just using the console, the examples provided should be easily readable and transferable to your preferred method. To keep things simple, we'll run with the assumption that you already have a VPC and subnet(s).

Make sure you have an AWS CLI profile configured for executing the Terraform code. I use a simple setup for the AWS provider like this:

```
# aws.tf
variable "aws_profile" {
  default = "<PROFILE>"
}
variable "aws_region" {
  default = "<REGION>"
}

provider "aws" {
  region  = "${var.aws_region}"
  profile = "${var.aws_profile}"
}

```


```
## main.tf

# ECR registry
resource "aws_ecr_repository" "app" {
  name                 = "app"
  image_tag_mutability = "IMMUTABLE"
}


# The task definition resource is created at this time just to reference it in the service.
# This will be continuously updated as part of application deployment.

# ECS Cluster

resource "aws_ecs_cluster" "app_ecs_cluster" {
  name = "ecs-app-cluster"
}

# Fargate Service
# Generate a random string to add it to the name of the Target Group
resource "random_string" "alb_suffix" {
  length  = 4
  upper   = false
  special = false
}

resource "aws_lb_target_group" "service" {
  name        = "tg-svc-${var.name}-${random_string.alb_suffix.result}"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    path = "/api/v1/health/status"
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb_listener" "service" {
  load_balancer_arn = var.load_balancer_arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.service.arn
  }
}

# resource "aws_ecs_task_definition" "service" {
#   family = "application"
#   container_definitions = var.task_container_definitions
# }

resource "aws_ecs_service" "service" {
  name            = var.name
  cluster         = var.cluster_id
  task_definition = var.task_definition_arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  network_configuration {
    assign_public_ip = true
    subnets          = var.subnets
  }

  load_balancer {
    container_port   = 8000
    target_group_arn = aws_lb_target_group.service.arn
    container_name   = "proxy" # TODO: Change this when task definition is in terraform
  }


  # Ignore count and task definition to allow scaling
  # and deployments outside terraform
  lifecycle {
    ignore_changes = ["desired_count", "task_definition"]
  }
}

```

## GitHub Deployment Workflow