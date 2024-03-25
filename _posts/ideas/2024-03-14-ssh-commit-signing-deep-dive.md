---
title: Deep Dive Into SSH Commit Signing
image:
  path: img/ssh-commit-signing.png
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

This command generates a new key-pair using the Ed25519 algorithm.
The `-C` flag is used to add a comment to the key. This is used when signing commits to identify the user who signed the commit.

After running the command, you will be asked where to save the key.
By default, the keys are saved in `~/.ssh/id_ed25519` for the private key and `~/.ssh/id_ed25519.pub` for the public key.

You can also specify a passphrase to protect the private key, but I will not be doing that in this example.
Personally, I have [1Password](https://developer.1password.com/docs/ssh) to manage my passwords, so that option is not something I need to explore.

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

The public key is a shorter string of characters that can be shared with anyone. It is used to verify that the data was signed with the private key.

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

In order to verify the signature, we need to add a `allowed_signers` file to specify which signatures are allowed.
To do so, create a file called `allowed_signers` and add the public key to it. All you need to do is move your email to the start of the line and add the public key at the end of the line.

For example, if your id_ed25519.pub file contains:

```
  ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDq04V5OobzsNHjBcavKcKKui2SByCnKSH0UYA/0Ntbm myuser@example.com
```

your allowed_signers file should contain:

```
  myuser@example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDq04V5OobzsNHjBcavKcKKui2SByCnKSH0UYA/0Ntbm
```

Next, let's verify the signature using the public key:

```bash
cat hello.txt | ssh-keygen -Y verify -n file -f allowed_signers -I myuser@example.com -s hello.txt.sig 
```

You should see this output:

```
Good "file" signature for myuser@example.com with ED25519 key SHA256:2aAXDGrsfr7AMN8uH/m2qtycnHuXoE6VFN9chBYcCCk
```

In the next section, we will explore the benefits of SSH commit signing by playing around with this verify command.

## Benefits of SSH commit signing

SSH commit signing provides several benefits:

1. **Integrity**: By signing your commits, you can ensure that the code has not been tampered with.
2. **Authentication**: By signing your commits, you can prove that you are the author of the code.
3. **Non-repudiation**: By signing your commits, you cannot deny that you authored the code.

Let's explore these benefits by signing a commit by contemplating the possibilities of what could go wrong if we didn't sign the commit.

#### Integrity
Let's say you are working on a project with a team of developers. One of the developers is a malicious actor who tampers with the code before committing it.
If you don't sign your commits, there is no way to verify that the code has been tampered with.
However, if you sign your commits, you can verify that the code has not been tampered with.

To demonstrate this, let's tamper with the file in the previous example:

```bash
echo "Hello, world! Tampered" > hello.txt
```

Now, let's verify the signature:

```bash
cat hello.txt | ssh-keygen -Y verify -n file -f allowed_signers -I myuser@example.com -s hello.txt.sig 
```

This time, you should see this output:

```
Signature verification failed: incorrect signature
Could not verify signature.
```

Nice! The signature verification failed, which gives us confidence that if our commits will be tampered with in any way, the verification will fail.
Any time the commit is changed (say, through a rebase), the commit has to be signed again, which requires the private key.

#### Authentication

Let's say you are working on a project with a team of developers. One of the developers is a malicious actor who commits code under your name.
If you don't sign your commits, there is no way to prove that you did not commit the code.
However, if you sign your commits, you can prove that you did not commit the code.

When you set up commit signing with GitHub (for example), you upload your public key. Similar to how we have the `allowed_signers` file, GitHub has a list of allowed signers.
When you sign a commit, GitHub can verify that the commit was signed with your private key, which includes your email address on GitHub.

To demonstrate this, using the same earlier example, let's change the signature in the `allowed_signers` file for `myuser@example.com` and try to verify the signature:

```bash
cat hello.txt | ssh-keygen -Y verify -n file -f allowed_signers -I myuser@example.com -s hello.txt.sig 
```

This time you should see this output:

```
known_principals:1: invalid key
Could not verify signature.
```

The verification failed because the signature in the `allowed_signers` file does not match the signature of the file, which means that the commit was not signed by the correct user.

#### Non-repudiation
This is similar to the authentication benefit, but it is more about proving that you did commit the code.
If you sign your commits, you cannot deny that you committed the code because the signature is unique to your private key.

## Conclusion

SSH commit signing is a powerful tool that provides integrity, authentication, and non-repudiation for your commits.
Although we explored signing an arbitrary file in this post, the same principles apply to signing commits.
I chose not to go into the details of setting up commit signing with Git because it is [well-documented and easy to set up](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits).

There are many other options to explore with ssh-keygen, but this understanding is definitely a step in the right direction.
