---
layout: post
title: "Mellotippet Status Update #4 - Prediction UI"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## My plan for the week

- Create a beautiful draggable UI element for the predictions page.
- Make it work to submit predictions for a heat with the new architecture.

## What I actually did this week

This week went very well! I accomplished what was planned, and then some. Here are the updates to the app:

- Users can make predictions using a draggable UI elements.
- Users can submit their prediction for a heat. 
- The songs and artists are fetched from the database.

The reason why it went so well this week was because it was much easier to create the draggable UI element than I had expected. I used [Draggable](https://api.flutter.dev/flutter/widgets/Draggable-class.html) and [DragTarget](https://api.flutter.dev/flutter/widgets/DragTarget-class.html) which worked very well. It was also a lot of fun to create something visual. 

Here is the prediction page in action:

<video width="240" height="480" controls>
  <source src="{{asset_path}}/prediction-page-ui.mp4" type="video/mp4">
</video>

I focused mostly on getting the functionality to work, there is a lot that does not follow the design yet. The image is hardcoded for now but that is easy to change. The toolbar and bottom nav bar don't match the design yet, I'll change that next week. Also, this is just a first version of the design - I expect improvements and changes in the future.

What's great is that the functionality works. The "Tippa" button to submit one's prediction is only enabled when there is a prediction for all spots, and the prediction is uploaded successfully to the database. With the prediction in the database the score is calculated correctly by the calculateScore firebase function when the result for the competition is added. Everything works perfectly!

## Next week

Soon I want to start uploading the app to Google Play Store and App Store, and start getting some testers, but I'll use next week to continue implementing the design. The designer will work on the design tomorrow and I look forward to see what we'll get :)

My plan for next week:
- Create a first version of the Score page.
- Make the toolbar match the design.
- Make the bottom nav bar match the design.

<br>

Score page design             |  Prediction page design
:-------------------------:|:-------------------------:
![Score page design]({{asset_path}}/score-page-design.png){:width="200px"}  |  ![Prediction page design]({{asset_path}}/prediction-page-design.png){:width="200px"}






