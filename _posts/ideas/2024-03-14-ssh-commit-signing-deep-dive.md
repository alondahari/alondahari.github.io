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

Think of those wax seals that were used to seal letters in the past. The private key is the stamp, and the public key is the wax seal.
Anyone can verify that the letter was sealed with your stamp by checking the wax seal.
However, only you can create the wax seal because only you have the stamp.

## Hands-on exploration

Let's explore ssh-keygen and see how it works. I will be using the following command to generate a new key-pair:

```bash
ssh-keygen -t ed25519 -C "myuser@example.com"
```

This command generates a new RSA key-pair with a key length of 4096 bits.
The `-C` flag is used to add a comment to the key. This is used when signing commits to identify the user who signed the commit.

After running the command, you will be asked where to save the key.
By default, the keys are saved in `~/.ssh/id_ed25519` for the private key and `~/.ssh/id_ed25519.pub` for the public key.

Let's take a look at the private key:

```bash
cat ~/.ssh/id_ed25519
```

The private key is a long string of characters that should never be shared with anyone.
The public key, on the other hand, can be shared with anyone.
Let's take a look at the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Next, let's sign a file using the private key:

```bash
echo "Hello, world!" > hello.txt
ssh-keygen -Y sign -n file -f ~/.ssh/id_ed25519 hello.txt
```

You should see this output:

```
Signing file hello.txt
Write signature to hello.txt.sig
```

This command signs the file `hello.txt` using the private key `~/.ssh/id_ed25519`.

The `-n file` flag specifies that the namespace is `file`, which means that the input is a file.
Other options include `blob` for binary data and `raw` for raw data, or a custom namespace.
For commit signing, the namespace is `git`.

The output is a signature that can be used to verify that the file was signed with the private key.
As a default, the signature is saved to `hello.txt.sig`.
Let's look at the signature:

```bash
cat hello.txt.sig
```

It is a long string of characters that can be used to verify the signature.

In order to verify the signature, we need to add a known principals file to specify which signatures are allowed.

Next, let's verify the signature using the public key:

```bash
ssh-keygen -Y verify -n file -f ~/.ssh/id_ed25519.pub -I myuser@example.com -s hello.txt.sig hello.txt
```
