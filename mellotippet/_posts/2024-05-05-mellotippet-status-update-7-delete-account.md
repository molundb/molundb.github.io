---
layout: post
title: "Mellotippet Status Update #7 - Delete Account In-App"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## What's happened
 
Since Melodifestivalen 2024 ended I've been taking a hiatus from developing Mellotippet. The [last thing that happened](/blog/2024/02/13/mellotippet-status-update-6-mvp-rejections-&-hiatus-copy) was that I managed to create a working MVP that was rejected by both Google and Apple.

The app was rejected for the following reasons:

1. Copycat of Melodifestivalen
2. Minimum functionality
3. No in-app account deletion functionality
4. Tracking without consent

I intend to tackle them one by one to eventually get the app approved and available in the Play and App Stores, and ready for Melodifestivalen 2025.

## What's been done since the last update

### In-App Account Deletion

I started with the easiest one: "no in-app account deletion functionality". To solve this I added a button in the appBar menu for users to delete their accounts. I could've put the deletion logic in the app or in the backend. I think it would be slightly better if it was in the backend but I put it in the app since it was the easiest. If the logic or database gets more complicated I might move it to the backend.

I added a button "ta bort konto" in the appBar menu:

![menu button]({{asset_path}}/menu-button.png){:width="240px"}

When the button is pressed a confirmation popup is displayed: 

![confirmation]({{asset_path}}/confirmation.png){:width="240px"}

If the user confirms deleting their account a loading is spinner is displayed while the account and associated information in the database gets deleted.

![loading]({{asset_path}}/loading.png){:width="240px"}

If the deletion is successful the user is logged out.

![login]({{asset_path}}/login.png){:width="240px"}

It would be nice to show some confirmation screen, but for now I think this solution is good enough. 

### Sign up fixes

While implementing in-app account deletion I noticed and fixed the following issues: 

- It was not possible to log out right after signing up.
- There was no loading indication while signing up.


## What's next
First of all I need to add error handling and tests of the new account deletion functionality. After that, the next major thing to do is to remove tracking or ask for consent and use app tracking transparency.


Thanks for reading. As always, you can find the project on github [here](https://github.com/molundb/mellotippet).

