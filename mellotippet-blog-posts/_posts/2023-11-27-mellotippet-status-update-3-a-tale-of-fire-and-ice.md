---
layout: post
title: "Mellotippet Status Update #3 - A Tale of Fire and Ice"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

Fire as in Firebase, and ice as in [freezed](https://pub.dev/packages/freezed). ;)

## This was my plan for the week:

- Make architectural improvements to the flutter code, like using [freezed](https://pub.dev/packages/freezed) for data classes.
- Set proper Firebase Security Rules and unit test them.
- Start looking into how to make a beautiful draggable UI element, which will be needed for the predictions page.

## What I actually did this week

It went well although I did not completely follow the plan. What I ended up doing was:

- I added tests for my [idempotent calculateScore firebase function](https://github.com/molundb/mellotippet/blob/main/functions/src/score-controller.ts).
- I fixed bugs discovered while adding [tests](https://github.com/molundb/mellotippet/blob/main/functions/test/score-controller.spec.ts) for calculateScore.
- I added [freezed](https://pub.dev/packages/freezed) and used it for all my models and riverpod state classes. Data classes yay <3

Adding tests for calculeScores after making it idempotent took a lot of effort, but it made me discover bugs and was definitely worth it. An issue I ran into was that all of a sudden all the tests started failing. How strange... 

After some investigation I realised that it was because I had reached the usage limits of my firebase test project:

![firebase test project usage limit reached]({{asset_path}}/usage-and-billing.png){:width="800px"}

I looked into how much it would cost to upgrade to a blaze plan and keep going. My project is using eur3 which means the cost would only be $0.02 per 100.000 deletes. That's nothing!

![passing tests]({{asset_path}}/pricing-eur3.png){:width="800px"}

Still, I waited until the next day and when I ran the tests they all passed. Nice!

![passing tests]({{asset_path}}/unit-tests.png){:width="800px"}

I expect to not have to work on calculateScores much more in the future (let's see if that holds true). The designer has worked hard and I'm thrilled to start implementing her beautiful design, which is what I will do this week. 

## My plan for next week:

- Create a beautiful draggable UI element for the predictions page.
- Make it work to submit predictions for a heat with the new architecture.
