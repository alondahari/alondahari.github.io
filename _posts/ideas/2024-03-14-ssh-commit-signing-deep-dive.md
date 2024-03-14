---
title: Deep Dive Into SSH Commit Signing
categories:
  - Git
  - Security
---

I recently worked on an implementation of SSH commit signing for the [Git Patch Stack open source project](https://git-ps.sh/).
While I was able to figure out what was needed to be done in order for me to complete the feature, it left me with the desire to learn more about the subject.

Some tools on my software development belt are like a hammer. I know how to use them, I know exactly how they work and how they were built.
However, some tools on my belt are more like a self-leveling cross-line laser level; I know how to set up and use them, but I have no idea how they work or built.
The aim of writing this blog post is to make SSH commit signing more like a hammer on my belt.
