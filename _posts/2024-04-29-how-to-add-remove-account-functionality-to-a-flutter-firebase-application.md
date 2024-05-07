---
layout: post
title: "How to Add Remove Account Functionality to a Flutter Firebase Application"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## Introduction

In my hobby application called [Mellotippet](https://github.com/molundb/mellotippet) users can create accounts. The first time I tried to upload Mellotippet to the Play and App stores it got [rejected](/blog/2024/05/05/mellotippet-status-update-7-delete-account) because there was no way for users to remove their accounts in the app.

![rejected by apple]({{asset_path}}/apple-rejected.png){:width="1200px"}


## What Remove Account Should Remove

Apple's guidelines for account deletion contain a lot of valuable information: [https://developer.apple.com/support/offering-account-deletion-in-your-app](https://developer.apple.com/support/offering-account-deletion-in-your-app).

"Offer to delete the entire account record, along with associated personal data"

- All data related to the user
- The login information (account record)


When deleting a user account you should run the deletions as a [set of atomic operations](https://firebase.google.com/docs/firestore/manage-data/transactions):

"Cloud Firestore supports atomic operations for reading and writing data. In a set of atomic operations, either all of the operations succeed, or none of them are applied."

Cloud Firestore supports two different ways of creating atomic operations: transactions and batched writes. 

Differences:
- In a batched write it's only possible to write documents; in a transactions you can write and *read* documents.
- A batched write will execute when the user's device is offline, a transaction will not.
- A batched write uses [less time and bandwidth](https://stackoverflow.com/a/70688794/2225594).

It's best to choose a batched write if you don't need to read documents.


## Outro

I hope this tutorial is helpful. The source code can be found here: [https://github.com/molundb/mellotippet](https://github.com/molundb/mellotippet).
