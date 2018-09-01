---
title: "What are your first steps when looking at a new codebase?"
date: 2017-11-22T00:33:22+01:00
draft: false
type: "post"
tags:
  - hbs-writing-series
  - getting-start-on-new-codebase
---

> When looking at a new codebase, the first thing I look out for is the docs for getting set up. If none exists, I ping the contact person to find out how to get set up. It’s important to always document what worked/didn’t work for me while getting setup because sometimes, even on a repo with documentation, the original developer(s) may have missed documenting a key component because they were writing the doc with loads of context and local libraries/packages that a developer new to the codebase might not have. Setting up can sometimes take forever, this is one of the reasons why I appreciate the concept of automating your development environment for a codebase as it gets complex beyond a simple Rails app setup.

> When I'm done setting up, I’ll update the readme with how I got set up so it's easier for the next developer. Updating the readme is good practice, but I sometimes forget to do this.
>
> Once my development environment is set up to run the code, I’ll run the tests. If all is well on my development and test environment, I like digging in to figure out what the major user journeys/flows for the application are. I will schedule a video call with the client to figure this out if they aren't obvious from navigating around. Knowing these helps me in 2 ways:
>
> 1) This helps me quickly get my local environment to match production as close as possible from CRUD-ing/manipulating items on the app (an added benefit is I get to catch as many bugs as I can from using the app as a real user would). 
>
> 2) It also helps me to compartmentalize the models, relationships, controllers, views, services, etc and how things fit together in my mind which is super important for me. I take lots of notes at this stage.
>
> If there’s a backlog, my preferred next step is usually to pick a teeny-weeny task and get that done just to dive in and get my feet wet.

This post first appeared in a writing series with my colleagues at Happy Bear Software.

[Read more about how we get productive on a Ruby on Rails codebase](https://www.happybearsoftware.com/how-we-start-work-on-a-rails-codebase)
