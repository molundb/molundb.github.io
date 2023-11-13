---
layout: post
title: "Mellotippet Status Update #1 - Force Upgrade Dialog & Changing the Project and Package Name"
date: 2023-11-12
---

This was my plan for the week:

- open source the project
- add force upgrade functionality
- set proper Firebase Security Rules and unit test them

I achieved a bit less than that. What I ended up doing was:

- adding force upgrade functionality
- renaming the package name and firebase projects to Mellotippet

Adding force upgrade functionality went pretty quickly; what took time was writing a blog post about it, which I intend to release in the upcoming days. The biggest difference in outcome compared to the plan was to rename the package name and firebase projects to Mellotippet. The first name of the project was Melodifestivalen Competition, and the second name was MelloPredix. There was a mix of these names used for my firebase projects, in the code, and for package names. I decided to tidy up and change the name to Mellotippet for everything. Since I have three firebase projects - prod, stage and test - this meant creating and configuring three new firebase projects. It feels good that it's done.

Right now I'm using stage for development. Maybe at some point I'll create a dev environment for development and use stage as a pre-release testing environment, but I haven't seen the need for that yet.

### My plan for next week:

- Open source the project
- Set proper Firebase Security Rules and unit test them
- Make improvements to my firebase functions, like setting the region and making them idempotent
- Make architectural improvements to the flutter code, like using freeze package for data classes
- Get the design from my designer, and finally start implementing it :)

I like writing these status update posts and plan to post one every Sunday.
