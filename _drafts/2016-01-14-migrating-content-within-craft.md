---
layout: post
title:  "Migrating content within Craft"
date:   2016-01-14 10:00:00
author: Josh Angell
---

Recently I had to migrate content from one section in Craft to another, complete with content. There was a bunch of content in a Matrix that was the biggest issue and a bunch of other non-relational fields. I figured I would just show you all what I did, in case it helps anyone else out!

First I created two tasks - one to get the elements that needed migrating and group them into chunks and the other to run the actual migration on each of them.
