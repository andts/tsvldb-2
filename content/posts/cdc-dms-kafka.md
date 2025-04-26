+++
title = "Building a Change Data Capture Pipeline with AWS and Kafka"
description = "This blog post provides a step-by-step guide on how to build and configure a change data capture pipeline that captures all changes in a database and writes them into an audit log table in a useful format. The post covers the architecture of the pipeline, services used, and tips on how to set up each component."
date = "2024-04-28"
updated = "2024-04-28"
draft = true
+++

# Intro

In this post, I will try to write a detailed step-by-step guide on how to build and configure a change data capture
pipeline that captures all changes in a database and writes them into an audit log table in a useful format.  
{{ admonition(type="warning", title="WIP", text="Please note that this post is a work in progress, and some sections may
be incomplete or require further refinement. I will continue working on it until I feel it is done, and if it currently
doesn’t have the information you are looking for - be sure to come back later.") }}

# Use Case

The use case for this setup is obviously an auditing system that keeps track of all changes in a database. Where could
this be useful? One option is using it for security reasons, to track user activity. Another one - activity logging
with the possibility of reviewing changes in some data entity. I could imagine it even be helpful for debugging
purposes.

# Architecture

{{ admonition(type="info", title="Architecture", text="Note that this post is not just an exercise but comes from
personal working experience, so some architectural decisions were preconditions that couldn't be changed. There are many
ways such a pipeline could be implemented, but I will only be talking about what I had experience with.") }}  
The pipeline in this post is built in AWS using the following services:

- **Aurora** - a cloud **PostgreSQL**, the primary data source and destination
- **Database Migration Service, DMS** - a service that captures and outputs the changes in Aurora
- **Managed Streaming for Apache Kafka, MSK with MSK Connect** - a queue (or a stream-processing framework, if you will)
  used to process the stream of data changes. **Kafka Connect** (**MSK Connect** in AWS-land) is a service that will put
  the
  data into its final destination.
- **Kafka Streams App** - a minimal app that transforms events, so they are easy to use later. It will be built using
  **Spring Boot** with **Spring Kafka** and run in an **Elastic Container Service (ECS)** cluster.

# The Guide

## Step 0: Users and Permissions

## Step 1: Set up Aurora

Let's start by setting up the Aurora database. I will be doing this with a Pulumi script, as it automates a lot of boring stuff we wouldn't want to do ourselves, and also gives a nice reproducible thing for me to share here.

... Pulumi ...

## Step 2: Set up MSK Cluster

Next, let's create a Kafka cluster. We don't need anything fancy, a 2-node cluster will be more than enough.
This cluster will be the target of the CDC Task. It will get messages about changes in the DB, we will process them using streams,
and will then write them to the audit log table from the final topic.

... Pulumi ...

Also, let's configure access to the cluster from the outside internet (but be carefull not to leave it that way, I only do this so it would be easier to monitor the cluster and inspect the topics).

... Pulumi ...

## Step 3: Create MSK Connector

## Step 4: Configure DMS Task

Now we can configure the DMS Task that will be watching doing the Change Data Capture. As we are using Aurora PostgreSQL, DMS will be monitoring the logical replication log, and will send message into a kafka topic.

... Pulumi ...

## Step 5: Build a Kafka Streams App



## Step 6: Test the Pipeline

