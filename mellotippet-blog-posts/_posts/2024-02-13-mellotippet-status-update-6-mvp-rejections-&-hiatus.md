---
layout: post
title: "Mellotippet Status Update #6 - MVP, Rejections & Hiatus"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## What's happened
 
My last status update was about two months ago, on December 11. I didn't manage to follow my weekly rhythm, which I feel somewhat disappointed about. Nevertheless, I've worked a lot on the app, and the MVP is finished! Unfortunately, it got rejected by both Google and Apple.

## What's been done since the last update

- Major improvement of the Prediction Page.
- Made the Login and Sign Up Pages match the design.
- Added the possibility to log out.
- Greatly improved the code by creating a reusable PredictionPage and PredictionController to be used by the three types of competition.
- Implemented support for the Semifinal and Final competitions.
- Updated the launcher icons.
- Fetch artist images for current competition from Firebase Storage.
- Sent the app to Google and Apple for review (and got rejected).

I've improved the app substantially and made a fully functional MVP. Unfortunately, the first heat of Melodifestivalen was on February 3, and I wasn't able to get the app approved by neither Google nor Apple before that. It's still not available on either app store. Thankfully, friends and family are using the app as internal testers, and it's working well. 

## The current state of the app

The design of the Prediction Page has improved considerably.

Prediction Page (not predicted)                |  Prediction Page (predicted)
:-------------------------:|:-------------------------:|:-------------------------:
![Prediction Page (not predicted)]({{asset_path}}/prediction-page-not-predicted-with-artist.png){:width="200px"}  |  ![Prediction Page (predicted)]({{asset_path}}/prediction-page-predicted-with-artist.png){:width="200px"}

Unfortunately, there was a setback related to the artist images. I had taken them from [https://media.melodifestivalen.se/](https://media.melodifestivalen.se/), and I realised that their copyright stated that the images couldn't be used for my purpose. I sent an email to ask if they could make an exemption, to which they replied something to the effect of "Great app, but sorry, no". Thus, I had to remove the artist images and the Prediction Page now looks like this:

Prediction Page (not predicted)                |  Prediction Page (predicted)
:-------------------------:|:-------------------------:|:-------------------------:
![Prediction Page (not predicted)]({{asset_path}}/prediction-page-not-predicted.png){:width="200px"}  |  ![Prediction Page (predicted)]({{asset_path}}/prediction-page-predicted.png){:width="200px"}

I've also updated the Log In and Sign Up Pages to match the design.

Log In Page |         Sign Up Page
:-------------------------:|:-------------------------:|:-------------------------:
![Log In Page]({{asset_path}}/log-in-page.png){:width="200px"} |  ![Sign Up Page]({{asset_path}}/sign-up-page.png){:width="200px"}

The app and score calculation work as they should.

## Why the app was rejected

The following are the reasons the app was rejected:

1. Copycat of Melodifestivalen
2. Minimum functionality
3. No in-app account-deletion functionality
4. Tracking without consent

I will discuss them one by one.

**1. Copycat of Melodifestivalen**

This is the one I understand the least. There is an official app of Melofestivalen called Melodifestivalen that also has a prediction functionality, but it is completely different from mine. My plan to tackle this is to add more functionality and hope that resolves the issue.

**2. Minimum functionality**

Right now it's only possible to do the following in the app:

- sign up
- log in
- log out
- see your score
- predict the current competition
- read the rules for the different types of competition

That was the plan for the MVP, but it's not enough for Apple. My plan to tackle this is to add group functionality, and if that's not enough, to keep adding functionality until the app is approved. It'll be interesting to see when that point is reached.

**3. No in-app account-deletion functionality**

This one is pretty straightforward: there needs to be a way for users to delete their accounts from within the app. I'll simply add this functionality.

**4. Tracking without consent**

This one is also pretty straightforward. Either I need to remove the tracking, including crash reporting, or I need to ask for users' consent. I am not sure which approach to take here. Short-term the simplest is to not track, but long-term tracking is much better. I'll probably choose the long-term solution.

## The Future of Mellotippet
Because of other priorities I am unsure if I will work on Mellotippet anymore before the final on March 9. I don't expect to work on Mellotippet for quite some time after March 9, but I do intend to have a fully functioning app available in the app stores before next year.

The most pressing things to do are:
- Add in-app account-deletion functionality
- Add Group functionality

