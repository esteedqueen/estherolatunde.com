---
title: "Migrate Rails App From Heroku to AWS Elastic Beanstalk"
date: 2018-09-01
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

In this article, I will document the process of how I helped a small startup with tens of thousands of users and data migrate their Ruby on Rails application from [Heroku](https://heroku.com/) to [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) with no loss of data and service.

The application I migrated was a typical production Rails stack: Ruby on Rails backend/API (production and staging environments with Android, iOS and web clients), PostgreSQL database, Sidekiq for processing background jobs, Redis Cluster, AWS S3, A couple of Node microservices and custom domain names with wildcard SSL certificate. It was setup on Heroku with a couple of performance-m(web) and standard(worker) dynos and a few heroku add-ons.

I'll cover the following:

  - [How to Setup and Deploy a Rails app on AWS Elastic Beanstalk](#How-to-Setup-and-Deploy-a-Rails-app-on-AWS-Elastic-Beanstalk)
  - [How to Setup a new Database and also Migrate Existing PostgreSQL Database from Heroku to AWS RDS](#Database-Setup-and-Migration)
  - [How to Setup Sidekiq on AWS Elastic Beanstalk to run Background Jobs on Elastic Cache Redis Cluster](#Setup-Sidekiq-on-ElasicCache-Cluster) (coming soon)
  - How to point the app on AWS Elastic Beanstalk to a custom domain and configure SSL for the app using AWS Certificate Manager (coming soon)
  - How to Access the Rails console on AWS Elastic Beanstalk (coming soon)
  - Elastic Beanstalk shortcuts (equivalents of what developers love on Heroku) (coming soon)

I'll also include how to manage multiple environments (staging and production) and how to handle environment specific changes on Elastic Beanstalk.

For an application with a production environment that is actively used, it's advisable that you complete a full migration with a functioning staging environment first. Test the newly migrated staging application for a few days. Ensure the database, services, background job, logs, etc. work as they should before replicating the setup for the production environment.

This guide can also double as a how-to guide to deploy a new Rails app to AWS Elastic Beanstalk. You can just choose the parts that are relevant to you. So, let's begin!

# How to Setup and Deploy a Rails app on AWS Elastic Beanstalk {#How-to-Setup-and-Deploy-a-Rails-app-on-AWS-Elastic-Beanstalk}

## Step 1: Install the AWS Elastic Beanstalk CLI via Homebrew

```bash
brew update

brew install aws-elasticbeanstalk
```
OR if you're using a Linux or a Windows, you can install the AWS Elastic Beanstalk CLI via pip. Note: You must have pip and a supported version of Python install. I think Python is also a prerequesite on a Mac too.

```bash
pip install awsebcli --upgrade --user
```

See [doc](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html) for more details.

## Step 2: Initialize AWS Elastic Beanstalk for your application.
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

For the purpose of this article, my choices above were used for a sample application called [guestbook](https://github.com/esteedqueen/guestbook/pull/1). This sets up an environment _guestbook_ in EU (Ireland) region, using a Ruby 2.5 using Puma server and lets you access your EC2 instance using ‘eb ssh’. Your choices/settings should be based on what would work best for the application you're about to deploy.

Running through the above steps generates a `.elasticbeanstalk/` folder in the root folder of your application and modifies your `.gitignore` to ignore the `.elasticbeanstalk/` folder and its contents from being commited to git.

* Protip 1: If you made a mistake in any of the `eb init` steps above, you can just delete the `.elasticbeanstalk/` and run `eb init` again.

* Protip 2: When selecting a platform version above, it's important to note that although Elastic Beanstalk refers to the versions as `x.x`, I think it only supports the latest stable release for that particular version. E.g. if you chose version `2.3` and  your applications's ruby version is `2.3.1`, your deployment will not run successfully and you will need to upgrade your application to the latest `2.3` stable release which is `2.3.7`(at this time) to be able to successfully deploy this. You can always refer to the [Elastic Beanstalk Ruby Supported Platforms](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts.platforms.html#concepts.platforms.ruby) to keep updated with this as it [always changes](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/platform-history-ruby.html). Weird, I know. It's even annoying that the error message in the EB deployment logs doesn't make this clear.


## Step 3: Deploy the application

Let's create the staging and production environments:

Staging:

```bash
eb create guestbook-staging
```

Production:

```bash
eb create guestbook-production
```

When you run `eb create environment-name`, Elastic Beanstalk sets *everything* up for you, EC servers, loadbalancers, S3 buckets, security groups, etc. This will take some time and the logs will look like this:

```bash
Creating application version archive "app-fff-blah-blah-12345".
Uploading guestbook/app-fff-blah-blah-12345.zip to S3. This may take a while.
Upload Complete.
Environment details for: guestbook-staging
  Application name: guestbook
  Region: eu-west-1
  Deployed Version: app-fff-blah-blah-12345
  Environment ID: e-9yecjt96ay
  Platform: arn:aws:elasticbeanstalk:eu-west-1::platform/Puma with Ruby 2.5 running on 64bit Amazon Linux/2.8.0
  Tier: WebServer-Standard-1.0
  CNAME: UNKNOWN
  Updated: 2018-03-05 20:36:26.984000+00:00
Printing Status:
...
```

The applications we just created will not successfully build due to some errors as we haven't set up the `SECRET_KEY_BASE` environment credentials.

## Step 4: Set environment variables/credentials

Option 1: Via the dashboard

- Login to AWS console
- Select Elastic Beanstalk
- Go to the AWS region you select,
- Click on the applications you just created, click configuration and set the environment variables on the dashboard.

OR

Option 2: Set the env vars via the command line using the `eb setenv` command:

```
eb setenv SECRET_KEY_BASE=$(rails secret)
```

Now you should be able to access the applications via the elasticbeanstalk application URLs by running `eb open environment-name` but we need to continue with creating the database before we can get fully functioning applications.


# How to Setup a new Database and Migrate Existing PostgreSQL Database from Heroku to AWS RDS {#Database-Setup-and-Migration}

## Setup

1. Create an RDS instance on AWS

Login to the AWS console to create an RDS instance --> Select the PostgreSQL engine -> Specify DB Details (postgres version, instance name, username and password) --> the rest of the configuration details.

A few things to note:

- The RDS instances should be in the same region as your Elastic Beanstalk application

- Ensure you set the RDS instance to be publicly accessible, use the default VPC Security Group or create a new Security Group and then set the TCP port 5342 to be accessible anywhere.

Provisioning of the database instance will take a few minutes to launch.

2. Database configuration

Update `config/database.yml` with production and staging RDS credentials.

```
# config/database.yml
...

production:
  <<: *default
  adapter: postgresql
  encoding: unicode
  database: <%= ENV['RDS_DB_NAME'] %>
  username: <%= ENV['RDS_USERNAME'] %>
  password: <%= ENV['RDS_PASSWORD'] %>
  host: <%= ENV['RDS_HOSTNAME'] %>
  port: <%= ENV['RDS_PORT'] %>

staging:
  <<: *default
  adapter: postgresql
  encoding: unicode
  database: <%= ENV['RDS_DB_NAME'] %>
  username: <%= ENV['RDS_USERNAME'] %>
  password: <%= ENV['RDS_PASSWORD'] %>
  host: <%= ENV['RDS_HOSTNAME'] %>
```

Then update the environment credentials with database configs.

Your .env file should now look like similar to this:

```
# .env.staging

BUNDLE_WITHOUT=test:development
RACK_ENV=staging
RAILS_ENV=staging
RAILS_SKIP_MIGRATIONS=false
RAILS_SKIP_ASSET_COMPILATION=false
SECRET_KEY_BASE=blahblahblah12345690
RDS_USERNAME=username
RDS_DB_NAME=db-name-staging
RDS_PASSWORD=password
RDS_PORT=5432
RDS_HOSTNAME=db-name-staging.blahsblahs.eu-west-1.rds.amazonaws.com
```

* Protip 3:

To set multiple environment variables contained in a .env file at once via the command line:

```bash
eb setenv `cat .env.production`

OR

eb setenv `cat .env.staging`

```
This depends on the .env file name and the elasticbeanstalk environment you're currently on.


* Protip 4: You can switch between environments using the `eb use environment-name` command to when you want make environment specific changes.

## Migrate Exisiting Database from Heroku to RDS

1. Turn on maintenance mode on Heroku to prevent further write to the database.

```bash
heroku maintenance:on -a <app_name>
```

2. Capture backup on Heroku and dump backup to a local file

```bash
heroku pg:backups capture -a <app-name>
curl -o path-to-staging-heroku-db-dump-file `heroku pg:backups public-url -r staging`
```

3. Use pg_restore to dump the local database to the AWS RDS remote database

```bash
pg_restore --verbose --clean --no-acl --no-owner -h RDS_DB_HOST -U RDS_USER -d DB_NAME -j 8 path-to-staging-heroku-db-dump-file
```
You'll get a prompt to enter password. Enter the database password.

The restore process will take some time depending on the size of your database

Note: Depending on the version of postgresql on your database, you might get the error: `pg_restore: [archiver] unsupported version (1.13) in file header`. If you do get the error, [check this out for why](https://help.heroku.com/YNH1ZJUS/why-am-i-getting-pg_restore-archiver-unsupported-version-1-13-in-file-header-error-with-pg_restore). Upgrade your local postgresql version then try the above command again.


# How to Setup Sidekiq on AWS Elastic Beanstalk to run Background Jobs on Elastic Cache Redis Cluster {#Setup-Sidekiq-on-ElasicCache-Cluster}

(coming soon...)