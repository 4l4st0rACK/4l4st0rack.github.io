---
title: "OverTheWire Bandit: Level 31 → 32 Walkthrough"
date: 2026-10-22 18:25:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, git, git-push, gitignore, git-add]
description: OverTheWire Bandit level 31 to 32 walkthrough, pushing a file to a Git repository, bypassing a .gitignore with git add -f and configuring the author identity.
ref: bandit-31-32
serie: bandit
nivel: 32
---

So far, with Git, we only **read** repositories. This level flips it: we have to **write** to one, with a `push`. And along the way we hit two very real obstacles: a `.gitignore` trying to stop us from uploading the file, and the identity configuration Git requires in order to commit.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Clone the repo (from your machine)

As always, from your local machine. The credential for `bandit31-git` is bandit31's password.

```console
$ cd $(mktemp -d)
$ git clone ssh://bandit31-git@bandit.labs.overthewire.org:2220/home/bandit31-git/repo
$ cd repo
$ cat README.md
```

```text
This time your task is to push a file to the remote repository.

Details:
    File name: key.txt
    Content: 'May I come in?'
    Branch: master
```

The goal is clear: create a `key.txt` file with the content `May I come in?` and **push** it to the `master` branch.

## The first obstacle: the `.gitignore`

I create the file with any editor (`nano`, `vim`, whatever you use):

```console
$ echo 'May I come in?' > key.txt
```

But before adding it, I take a good look at the repo, including hidden files (`-a`):

```console
$ ls -la
...
.gitignore
$ cat .gitignore
*.txt
```

There's the trap: the `.gitignore` contains `*.txt`. A `.gitignore` tells Git which files to **ignore**, i.e. not to track or upload. The pattern `*.txt` ignores **all** files with a `.txt` extension... exactly our `key.txt`. If I do a normal `git add key.txt`, Git ignores it without complaint.

## Bypass the `.gitignore`: `git add -f`

To force Git to add an ignored file you use the `-f` (*force*) option:

```console
$ git add -f key.txt
```

The `-f` says "I know it's in the `.gitignore`, add it anyway". Without it, Git would skip it and it would never reach the commit.

## The second obstacle: the author identity

I try to commit:

```console
$ git commit -m "Create a new file"
```

And Git refuses:

```text
*** Please tell me who you are.
...
fatal: unable to auto-detect email address
```

Git **won't let you commit without knowing who you are**: every commit carries an author (name and email). On your normal machine this is already configured, but in a freshly cloned repo in a temporary directory, it isn't. I set it **just for this repo** (without `--global`):

```console
$ git config user.name bandit31
$ git config user.email bandit31@overthewire.org
```

> The value doesn't matter, it can be anything; Git just wants the field not to be empty. I use `git config` **without `--global`** so it only affects this repository and doesn't touch my machine's config.
{: .prompt-tip }

And now the commit goes through:

```console
$ git add -f key.txt
$ git commit -m "Create a new file"
```

## The solution: push

I push my commit to the remote's `master` branch:

```console
$ git push -u origin master
```

It asks for `bandit31-git`'s password and, after uploading the objects, the server validates what I sent and responds:

```text
remote: ### Attempting to validate files... ####
remote: Well done! Here is the password for the next level:
remote: [REDACTED]
...
 ! [remote rejected] master -> master (pre-receive hook declined)
```

There's bandit32's password. Notice the final detail: the push shows up as **`[remote rejected]`** / `pre-receive hook declined`. **That's normal and expected**, it's not a mistake on your part.

## Why does the push get "rejected" but still give you the password?

The server has a **`pre-receive` hook**: a script that runs on the remote **before** accepting a push and can accept or reject it. Here that hook checks whether you pushed `key.txt` with the correct content:

- If you did it right, it **prints the password**...
- ...but then **rejects the push anyway**, so as not to actually modify the game's repository.

In other words, the hook uses you purely as a validation mechanism: it rewards you with the password but doesn't let your commit into the repo. That's why you see the success message (`Well done!`) and the rejection (`remote rejected`) at the same time.

> Git hooks (`pre-receive`, `pre-commit`, `post-receive`...) are scripts that run automatically at specific points in Git's lifecycle. In the real world they're used to validate code, trigger CI/CD, prevent secrets from being pushed, etc. Here it's what validates your challenge.
{: .prompt-tip }

## Why this matters in security

Two ideas you'll genuinely run into:

- **`.gitignore` is not security.** It only prevents a file from being added by accident; it doesn't "protect" anything. `git add -f` bypasses it, and an ignored file can perfectly well be present on disk or have been pushed earlier. Never rely on `.gitignore` to "hide" secrets.
- **Server-side hooks** are a powerful control tool: validate what comes in, block pushes containing credentials (`pre-receive` + a secret scanner), enforce policies... Understanding that the server can run its own logic when it receives a push is key for both defense and offense.

## Moving to the next level

```console
$ sshpass -p '[REDACTED]' ssh bandit32@bandit.labs.overthewire.org -p 2220
```

> Git is over. The next level is one of the most legendary in Bandit: it drops you into an **"uppercase shell"** that converts everything you type to uppercase. You'll have to escape it with a very clever trick.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `.gitignore` | List of file patterns Git ignores (not security) |
| `git add -f file` | Force-add a file ignored by `.gitignore` |
| `git config user.name/email` | Set the author identity (without `--global`, this repo only) |
| `git push -u origin master` | Push your commits to the remote's `master` branch |
| `pre-receive` hook | Server-side script that validates or rejects a push |
| Deliberately rejected push | The hook gives you the password but doesn't let your commit in |

{% include series-nav.html %}
