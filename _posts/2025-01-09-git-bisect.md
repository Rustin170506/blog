---
title: 'Git Bisect'
layout: post

categories: post
tags:
- Git
- Tools
---

# Git Bisect

`git bisect` is a powerful tool for identifying the commit that introduced a bug or behavior change. By performing a binary search through the commit history, it efficiently narrows down the specific commit responsible for the issue. This tool can also automate the process, making it easier to pinpoint the problematic commit.

## Basic Usage

I use `git bisect` in two scenarios: when there is a bug that needs to be identified, or when I want to find the commit that fixed a bug. Each scenario requires a different approach.
