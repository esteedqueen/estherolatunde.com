---
title: "Migrate Rails App From Heroku to AWS Elastic Beanstalk"
date: 2018-03-29T10:59:33+01:00
draft: true
type: "post"
tags:
  - aws
  - elasticbeanstalk
  - rds
  - elasticcache
  - heroku
  - rails
  - postgresql
  - sidekiq
  - redis
  - background jobs
---

In this article, I document the process of how I helped a small startup with tens of thousands of users and data migrate their Ruby on Rails application from Heroku to AWS Elastic Beanstalk with no loss of data and service.

I love Heroku. I really do. The ease and speed at which it gets an indie developer/small startup from development to a live product is amazing! An added bonus is how it cuts out the dev ops overhead to the barest minimum effort until you scale to outgrow the platform or can no longer afford it.

I currently work on a project using a Cloud VPS with a mix of Ansible playbooks and Jenkins plugins for managing the deployment operations and I can't overstate my appreciation of Heroku.

The application I migrated was a typical production Ruby on Rails stack: Ruby on Rails backend/API (production and staging environments with Android, iOS and web clients), PostgreSQL database, Sidekiq for processing background jobs, Redis Cluster, AWS S3, A couple of Node Microservices and custom domain names with wildcard SSL certificate. It was setup on Heroku with a couple of performance-m(web) and standard(worker) dynos and a few heroku add-ons.

This is my attempt at a comprehensive guide that covers all aspect of the setup on AWS. I'll cover the following:
- How to Setup and Deploy a Rails app on AWS Elastic Beanstalk
- How to Manage and Deploy to multiple environments on AWS Elastic Beanstalk
- How to Migrate Existing PostgreSQL Database from Heroku to AWS RDS
- How to Setup Sidekiq on AWS Elastic Beanstalk to run Background Jobs on Elastic Cache Redis Cluster
- How to point the app on AWS Elastic Beanstalk to a custom domain and configure SSL for the app using AWS Certificate Manager
- How to Access the Rails console on AWS Elastic Beanstalk
- Elastic Beanstalk shortcuts (equivalents of what developers love on Heroku)

I will also try to highlight a few issues/gotchas I experienced in each of the steps above. However, if you encounter any issues, feel free to drop a note and I'll be happy to help.

This guide can also double as a how-to guide to deploy a new Rails app to AWS Elastic Beanstalk. You can just choose the parts that are relevant to you. So, let's begin

## How to Setup and Deploy a Rails app on AWS Elastic Beanstalk
