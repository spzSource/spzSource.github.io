---
layout: post
title: "Nuke.Build + Azure Pipelines: how to publish code coverage artifacts correctly"
date: 2020-05-08 22:56:00 +0300
categories: azure
tags: azure, pipelines, devops, nuke, build, coverage, publication, artifacts
---

On my current project we've started use [Nuke.Build](https://nuke.build/) for buld and deployment pipeline for our services. Througout last few months we did a great job for creating fully operation template for a service, allowing to spin up new services in a 10 minutes. At the same time there are still many issues that we need to address. On this article I want to describe one of the issue, which we've been trying to solve a long time.

