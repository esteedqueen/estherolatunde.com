---
title: "Migrate Rails App From Heroku to AWS Elastic Beanstalk"
date: 2018-03-29T10:59:33+01:00
draft: false
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

In this article, I will document the process of how I helped a small startup with tens of thousands of users and data migrate their Ruby on Rails application from Heroku to [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) with no loss of data and service.

The application I migrated was a typical production Ruby on Rails stack: Ruby on Rails backend/API (production and staging environments with Android, iOS and web clients), PostgreSQL database, Sidekiq for processing background jobs, Redis Cluster, AWS S3, A couple of Node microservices and custom domain names with wildcard SSL certificate. It was setup on Heroku with a couple of performance-m(web) and standard(worker) dynos and a few heroku add-ons.

I'll cover the following:

  - How to Setup and Deploy a Rails app on AWS Elastic Beanstalk
  - How to Migrate Existing PostgreSQL Database from Heroku to AWS RDS
  - How to Setup Sidekiq on AWS Elastic Beanstalk to run Background Jobs on Elastic Cache Redis Cluster
  - How to point the app on AWS Elastic Beanstalk to a custom domain and configure SSL for the app using AWS Certificate Manager
  - How to Access the Rails console on AWS Elastic Beanstalk
  - Elastic Beanstalk shortcuts (equivalents of what developers love on Heroku)

I will also try to highlight a few issues/gotchas I experienced in each of the steps above. However, if you encounter any issues, feel free to drop a note and I'll be happy to help.

This guide can also double as a how-to guide to deploy a new Rails app to AWS Elastic Beanstalk. You can just choose the parts that are relevant to you. So, let's begin!

## How to Setup and Deploy a Rails app on AWS Elastic Beanstalk

- Step 1: Install the AWS Elastic Beanstalk CLI via Homebrew

```bash
brew update

brew install aws-elasticbeanstalk
```
OR if you're using a Linux or a Windows, you can install the AWS Elastic Beanstalk CLI via pip. Note: You must have pip and a supported version of Python install. I think Python is also a prerequesite on a Mac too.

```bash
pip install awsebcli --upgrade --user
```

See [](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html) for more details.

- Step 2: Initialize AWS Elastic Beanstalk for your application.
From the home directory of the Rails app you want to deploy, run the command below:

```bash
eb init

```

This gives you an interactive prompt to set your app configuration. Here's what that will look like on your terminal.

```bash
Select a default region
1) us-east-1 : US East (N. Virginia)
2) us-west-1 : US West (N. California)
3) us-west-2 : US West (Oregon)
4) eu-west-1 : EU (Ireland)
5) eu-central-1 : EU (Frankfurt)
6) ap-south-1 : Asia Pacific (Mumbai)
7) ap-southeast-1 : Asia Pacific (Singapore)
8) ap-southeast-2 : Asia Pacific (Sydney)
9) ap-northeast-1 : Asia Pacific (Tokyo)
10) ap-northeast-2 : Asia Pacific (Seoul)
11) sa-east-1 : South America (Sao Paulo)
12) cn-north-1 : China (Beijing)
13) cn-northwest-1 : China (Ningxia)
14) us-east-2 : US East (Ohio)
15) ca-central-1 : Canada (Central)
16) eu-west-2 : EU (London)
17) eu-west-3 : EU (Paris)
(default is 3): 4

Select an application to use
1) My First Elastic Beanstalk Application
2) [ Create new Application ]
(default is 2): 2

Enter Application Name
(default is "guestbook"):
Application guestbook has been created.

It appears you are using Ruby. Is this correct?
(Y/n): y

Select a platform version.
1) Ruby 2.5 (Puma)
2) Ruby 2.5 (Passenger Standalone)
3) Ruby 2.4 (Puma)
4) Ruby 2.4 (Passenger Standalone)
5) Ruby 2.3 (Puma)
6) Ruby 2.3 (Passenger Standalone)
7) Ruby 2.2 (Puma)
8) Ruby 2.2 (Passenger Standalone)
9) Ruby 2.1 (Puma)
10) Ruby 2.1 (Passenger Standalone)
11) Ruby 2.0 (Puma)
12) Ruby 2.0 (Passenger Standalone)
13) Ruby 1.9.3
(default is 1): 1

Note: Elastic Beanstalk now supports AWS CodeCommit; a fully-managed source control service. To learn more, see Docs: https://aws.amazon.com/codecommit/
Do you wish to continue with CodeCommit? (y/N) (default is n): n

Do you want to set up SSH for your instances?
(Y/n): y

Type a keypair name.
(Default is xx):
If you already have a keypair set OR

1) [ Create new KeyPair ]
(default is 1)
```

For the purpose of this article, my choices above were used for a sample application called [guestbook](). This sets up an environment _guestbook_ in EU (Ireland) region, using a Ruby 2.3 using Puma server and lets you access your EC2 instance using ‘eb ssh’. Your choices/settings should be based on what would work best for the application you're about to deploy.

Running through the above steps generates a `.elasticbeanstalk/` folder in the root folder of your application and modifies your `.gitignore` to ignore the `.elasticbeanstalk/` folder and its contents from being commited to git.

* Gotcha 1: If you made a mistake in any of the `eb init` steps above, you can just delete the `.elasticbeanstalk/` and run `eb init` again.

* Gotcha 2: When selecting a platform version above, it's important to note that although Elastic Beanstalk refers to the versions as `x.x`, I think it only supports the latest stable release for that particular version. E.g. if you chose version `2.3` and  your applications's ruby version is `2.3.1`, your deployment will not run successfully and you will need to upgrade your application to the latest `2.3` stable release which is `2.3.7`(at this time) to be able to successfully deploy this. You can always refer to the [Elastic Beanstalk Ruby Supported Platforms](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts.platforms.html#concepts.platforms.ruby) to keep updated with this as it [always changes](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/platform-history-ruby.html). Weird, I know. It's even annoying that the error message in the EB deployment logs doesn't make this clear.

- Step 3: Deploy your application

Let's create the staging and production environments:

Staging:

```bash
eb create guestbook-staging
```

Production:

```bash
eb create guestbook-production
```

When you run `eb create app-name`, Elastic Beanstalk sets *everything* up for you, EC servers, loadbalancers, S3 buckets, security groups, etc.

The logs will look like this:

```bash
Creating application version archive "app-ffb8-180605_213621".
Uploading guestbook/app-ffb8-180605_213621.zip to S3. This may take a while.
Upload Complete.
Environment details for: guestbook-staging
  Application name: guestbook
  Region: eu-west-1
  Deployed Version: app-ffb8-180605_213621
  Environment ID: e-9yecjt96ay
  Platform: arn:aws:elasticbeanstalk:eu-west-1::platform/Puma with Ruby 2.5 running on 64bit Amazon Linux/2.8.0
  Tier: WebServer-Standard-1.0
  CNAME: UNKNOWN
  Updated: 2018-06-05 20:36:26.984000+00:00
Printing Status:
...
```

At this point, the






