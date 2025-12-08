---
title: "I set up CI/CD for my blog!"
date: 2025-12-08T09:15:00+03:00
draft: false
tags: ["devops", "github-actions", "hugo"]
categories: ["automation"]
---

Hello everyone! 👋

I finally automated the deployment of this blog.

### How it works:
1. I write a post in Markdown locally.
2. Run `git push`.
3. **GitHub Actions** connects via SSH.
4. Updates code and rebuilds static files via Docker.

Process takes **12 seconds**. 🚀
