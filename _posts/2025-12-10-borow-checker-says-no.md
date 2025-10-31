---
title: 'Borrow checker says “No”! An error that scares me every single time!'
date: 2025-10-31 00:00:00
---

It’s Halloween and I have just been caught out by a spooky borrow checker error that caught me by surprise. It feels as though it is the single most time consuming issue to fix and always seems to catch me unaware. The issue in particular is “cannot borrow x immutably as it is already borrowed mutably” - it manifests itself in different ways under different circumstances, but I find myself hitting it often when refactoring. It happened again recently so I did some investigating and thought I would discuss it in more detail.

The issue last hit me when I was refactoring some code in my graphics engine [hotline](https://github.com/polymonster/hotline), I have been creating some content on YouTube and, after a little bit of a slog to fix the issue, I recorded a video of me going through the scenario of how it occurred and some patterns to use that I have adopted in the past to get around it. You can check out the video if you are that way inclined, the rest of this post will mostly echo what is in the video, but it might be a bit easier to follow code snippets and description in text.  

