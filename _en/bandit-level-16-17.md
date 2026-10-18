---
title: "OverTheWire Bandit: Level 16 → 17 Walkthrough"
date: 2026-10-18 23:15:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, ssl, tls, ncat, nmap, port-scanning, bash, ssh]
description: OverTheWire Bandit level 16 to 17 walkthrough, finding the only SSL/TLS service among a thousand ports that returns the credentials, with a bash loop and with nmap.
ref: bandit-16-17
serie: bandit
nivel: 17
---

In the [previous level](/en/posts/bandit-level-15-16/) we knew exactly which port to talk to. Not anymore: the service is **somewhere between port 31000 and 32000**, it speaks SSL/TLS and there are decoys too. This level is a small introduction to something you'll do in every pentest: **finding out which ports are open and which service runs on each one**.

> Level passwords and keys are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit16@bandit.labs.overthewire.org -p 2220
bandit16@bandit:~$ export TERM=xterm
```

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit17.html) says the credentials for the next level can be retrieved by **submitting the current level's password to a port on localhost between 31000 and 32000**. First you need to find which ports have a server listening and, among those, which ones speak SSL/TLS. Only one of them gives the next credentials; the others simply send back whatever you send them.

```console
bandit16@bandit:~$ ls
bandit16@bandit:~$
```

Empty home again. Everything happens on the network.

## The solution: a loop that tries all 1001 ports

From the previous level we already know how to send the password to a TLS service with `ncat --ssl`. The difference is that we don't know the port now, so I try all of them with a bash `for` loop:

```console
bandit16@bandit:~$ for port in {31000..32000}; do ncat --ssl localhost ${port} < /etc/bandit_pass/bandit16 2>/dev/null; done
```

Among other things, an **SSH private key** shows up on screen:

```console
[REDACTED: SSH private key]
```

Breaking the command down piece by piece:

- `{31000..32000}` is a bash **brace expansion**: it generates every number from 31000 to 32000 (1001 values). The loop walks through them one by one, storing each number in the `port` variable.
- `ncat --ssl localhost ${port}` tries to open a TLS connection to that port, just like in the previous level.
- `< /etc/bandit_pass/bandit16` feeds the password through standard input, no copy-pasting.
- `2>/dev/null` throws errors away. And there will be a lot of them.

Why so many errors? On each port, one of these things can happen:

| Port situation | What `ncat --ssl` does |
| --- | --- |
| Closed (nothing listening) | Fails to connect (connection refused) |
| Open, no SSL/TLS | Connects over TCP but the TLS handshake fails |
| Open, SSL/TLS, decoy | Sends back exactly what you sent (your own password) |
| Open, SSL/TLS, the right one | Sends back the level 17 credentials |

The first two cases write errors to stderr, and since the vast majority of the 1001 ports are closed, without `2>/dev/null` the screen would be a wall of messages. Silencing them leaves only what the real TLS services answer.

> This loop works, but it has two downsides: it's slow (1001 connection attempts) and **it doesn't tell you which port answered what**. You'll see your own password repeated (the decoys) and the key, without knowing where each one came from. We'll improve it in the next section.
{: .prompt-info }

## Using the key to log in as bandit17

What the service returns isn't a password but an **SSH private key**, like the one we used in level 13 → 14. I take it to my own machine: I copy the whole key block (including the `-----BEGIN ...-----` and `-----END ...-----` lines) into a local file, for example `bandit17.key`.

Before using it, I change its permissions:

```console
$ chmod 600 bandit17.key
$ ssh -i bandit17.key bandit17@bandit.labs.overthewire.org -p 2220
```

Why `chmod 600`? SSH is very strict about private keys: if anyone other than you can read the file, **it refuses to use it** and warns you that the key is unprotected. `600` means read and write for the owner only (`rw-------`), which is exactly what it wants.

> Once you're in as `bandit17`, run `cat /etc/bandit_pass/bandit17` and save the password. Next time you can log in with `sshpass` as usual, without depending on the key file.
{: .prompt-tip }

## Other ways to do it

### 1. A loop that tells you which port answers

We can improve the loop so it only shows ports that return **something different** from what we sent, in other words, so it ignores the decoys:

```console
bandit16@bandit:~$ pass=$(cat /etc/bandit_pass/bandit16)
bandit16@bandit:~$ for port in {31000..32000}; do out=$(timeout 3 ncat --ssl localhost $port <<< "$pass" 2>/dev/null); if [ -n "$out" ] && [ "$out" != "$pass" ]; then echo "[+] Port $port"; echo "$out"; fi; done
```

What's new here:

- `pass=$(cat ...)` stores the password in a variable (**command substitution**).
- `out=$(...)` captures each port's reply in another variable instead of dumping it straight to the screen.
- `<<< "$pass"` is a **here-string**: it feeds the variable's content through standard input.
- `timeout 3` kills `ncat` if a port hangs for more than 3 seconds, so the loop doesn't get stuck.
- `[ -n "$out" ]` checks the reply isn't empty, and `[ "$out" != "$pass" ]` that it isn't our own password echoed back.

The result is a single `[+] Port ...` line followed by the key. Much cleaner.

### 2. The pentester way: `nmap`

In real life you wouldn't try ports one by one with a loop: you'd use a port scanner. `nmap` is installed on the Bandit server.

First, find which ports in the range are open:

```console
bandit16@bandit:~$ nmap -p 31000-32000 localhost
```

This gives you a short list of open ports. Then I ask `nmap` to identify which service runs on each one with `-sV` (version detection), only on those ports:

```console
bandit16@bandit:~$ nmap -sV -p <open_ports> localhost
```

With the ports separated by commas (for example `-p 31xxx,31yyy`). In the `SERVICE` column you'll see which ones are prefixed with `ssl/`: those speak TLS. Among them, the ones `nmap` identifies as `echo` are the decoys (they return whatever they receive), and the one left is your target. Then you just send the password to that port:

```console
bandit16@bandit:~$ ncat --ssl localhost <port> < /etc/bandit_pass/bandit16
```

> `-sV` is slower because `nmap` talks to each service to identify it. That's why you run a quick port scan first and then `-sV` only on the open ports. It's exactly the workflow you'll use on HTB machines or in a real pentest.
{: .prompt-tip }

### 3. `openssl s_client` against the right port

Once you know the port, the previous level's tool works too. Remember `-quiet` to avoid the interactive commands issue (`Q`, `R`, `k`, `K`) and so it doesn't close before receiving the reply:

```console
bandit16@bandit:~$ openssl s_client -connect localhost:<port> -quiet < /etc/bandit_pass/bandit16
```

## Moving to the next level

I exit bandit16 and log in as bandit17 with the key from my machine:

```console
bandit16@bandit:~$ exit
$ chmod 600 bandit17.key
$ ssh -i bandit17.key bandit17@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
| --- | --- |
| `for port in {31000..32000}; do ...; done` | Loop over a range of ports in bash |
| `{start..end}` | Brace expansion: generates a sequence of numbers |
| `2>/dev/null` | Discard errors (closed ports or no TLS) |
| `$(command)` / `<<< "$var"` | Capture output in a variable / feed a variable through stdin |
| `timeout N command` | Stop a command that takes longer than N seconds |
| `nmap -p 31000-32000 localhost` | Find which ports are open in a range |
| `nmap -sV -p ...` | Identify the service on each port (and whether it uses SSL/TLS) |
| `chmod 600 key` | The permissions SSH requires for a private key |
| `ssh -i key user@host -p 2220` | Log in with a private key instead of a password |

{% include series-nav.html %}
