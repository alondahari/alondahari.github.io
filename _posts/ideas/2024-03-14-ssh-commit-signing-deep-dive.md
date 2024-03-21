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

In this post, I will cover:

- SSH signing at a high level
- An exploration of ssh-keygen with hands-on examples
- The benefits of SSH commit signing

## How does signing work?

Without going into any implementation details, this is the general idea:

You create a key-pair with a signing program such as ssh-keygen.
Based on the options you provide, one private key and one public key are created.
The private key is yours to keep and never share. This key is used for signing any piece of data, such as a file or commit.
The public key can be used by anyone to verify that the data is signed with your private key, without the need for it.
