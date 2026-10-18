---
title: "OverTheWire Bandit: Level 15 → 16 Walkthrough"
date: 2026-10-18 22:55:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, ssl, tls, openssl, ncat, networking]
description: OverTheWire Bandit level 15 to 16 walkthrough, sending a password to an SSL/TLS encrypted service on localhost with ncat and openssl s_client.
ref: bandit-15-16
serie: bandit
nivel: 16
---

In the previous level we talked to a network service in plain text. This time it's the same idea, but the service is **encrypted with SSL/TLS**, and that changes the tools: `nc` no longer works because it doesn't "speak" TLS. Time to meet `ncat --ssl` and, above all, `openssl s_client`, a tool you'll use constantly in pentesting to inspect encrypted services.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit15@bandit.labs.overthewire.org -p 2220
bandit15@bandit:~$ export TERM=xterm
```

As always, `export TERM=xterm` so the remote terminal behaves properly with `clear`, colors and so on.

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit16.html) says the password for the next level can be retrieved by **submitting the current level's password to port 30001 on localhost using SSL/TLS encryption**.

First, let's see what's in the home directory:

```console
bandit15@bandit:~$ ls
bandit15@bandit:~$
```

Nothing. This level isn't about files, it's about networking. What I need is:

1. The current level's password (bandit15).
2. A tool that opens a **TLS** connection to `localhost:30001` and lets me send it text.

## Step 1: the current level's password

Each user's password lives in `/etc/bandit_pass/`, and each user can only read their own:

```console
bandit15@bandit:~$ cat /etc/bandit_pass/bandit15
[REDACTED]
```

> This only works because I'm logged in as `bandit15`. Try reading `/etc/bandit_pass/bandit16` and you'll get `Permission denied`: the file permissions only let its own user read it.
{: .prompt-info }

## Step 2: talking to the service over TLS

### Why doesn't `nc` work?

In the previous level, `nc localhost 30000` was enough: we opened a TCP connection and whatever we typed was sent as is. But the service on port 30001 expects the client to start a **TLS handshake** right after connecting: negotiate the version and cipher, receive the server certificate and agree on session keys. Only after that does data start flowing (encrypted).

The `nc` installed on the server can't do that handshake. If you send the password in plain text, the server receives bytes that aren't a valid TLS message and drops the connection. We need a client that performs the handshake for us and then lets us type as if it were a normal connection.

### The solution: `ncat --ssl`

`ncat` is the modern netcat that ships with Nmap, and one of its improvements is SSL/TLS support through the `--ssl` option:

```console
bandit15@bandit:~$ ncat --ssl localhost 30001
[REDACTED]
[REDACTED]
```

The first `[REDACTED]` line is bandit15's password, which I paste in. The server receives it, checks it's correct and answers with the password for **bandit16**.

What's happening under the hood:

- `ncat` opens a TCP connection to `localhost:30001`.
- With `--ssl` it performs the TLS handshake first.
- From then on, whatever I type (stdin) is encrypted and sent, and the server's reply is decrypted and printed to the screen (stdout).

> This service uses a self-signed certificate. `ncat --ssl` **doesn't verify the certificate by default**, which is why it connects without complaining. To require a valid certificate you'd use `--ssl-verify`, and here it would fail. In a lab that's fine, but in real life connecting without verifying the certificate exposes you to man-in-the-middle attacks.
{: .prompt-tip }

## Other ways to do it

### 1. `ncat` without pasting anything

Instead of running `cat` and copy-pasting, I can redirect the file straight into `ncat`'s input, just like we did with redirections in earlier levels:

```console
bandit15@bandit:~$ ncat --ssl localhost 30001 < /etc/bandit_pass/bandit15
```

Or with a pipe, which does the same thing:

```console
bandit15@bandit:~$ cat /etc/bandit_pass/bandit15 | ncat --ssl localhost 30001
```

This avoids copy mistakes (an extra space, a stray newline...) and the password never goes through your clipboard.

### 2. `openssl s_client` (the "official" tool)

`openssl s_client` is a generic TLS client. It's what people use in the real world to inspect a server's certificate, check which TLS versions it accepts, or talk by hand to encrypted services (HTTPS, SMTPS, IMAPS...).

```console
bandit15@bandit:~$ openssl s_client -connect localhost:30001
```

You'll see a lot of output: the certificate chain, the negotiated TLS version, the cipher... and a warning that the certificate couldn't be verified (it's self-signed). Then it waits: paste the password, press Enter and you get level 16's password back.

But there's a catch, and it's exactly the hint the level gives about `DONE`, `RENEGOTIATING` or `KEYUPDATE`. In interactive mode, `s_client` treats some letters **at the start of a line** as commands (the *CONNECTED COMMANDS* section of `man s_client`):

| Letter at start of line | What it does |
| --- | --- |
| `Q` | Closes the connection (you'll see `DONE`) |
| `R` | Renegotiates the TLS session (`RENEGOTIATING`) |
| `k` / `K` | Updates the keys (`KEYUPDATE`) |

If the password you paste starts with one of those letters, `s_client` doesn't send it: it runs the command instead. The fix is to disable those commands with `-quiet` (or `-ign_eof`):

```console
bandit15@bandit:~$ openssl s_client -connect localhost:30001 -quiet
```

### 3. `openssl` as a one-liner

Combining the above with a redirection:

```console
bandit15@bandit:~$ openssl s_client -connect localhost:30001 -quiet < /etc/bandit_pass/bandit15
```

Here `-quiet` does two things: it suppresses the certificate and session output, and it implicitly enables `-ign_eof`. That second part matters: when input comes from a file or a pipe, it ends almost immediately (EOF). Without `-ign_eof`, `s_client` would close the connection as soon as it finishes reading the file, sometimes **before the server's reply arrives**, and you wouldn't see the password. With `-ign_eof` it ignores the end of input and waits for the server to answer.

> If the certificate warning on stderr bothers you, add `2>/dev/null` at the end to discard it, like we did with `find` in earlier levels.
{: .prompt-tip }

## Moving to the next level

```console
bandit15@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit16@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
| --- | --- |
| `cat /etc/bandit_pass/bandit15` | Read the current user's password |
| SSL/TLS | Encryption layer on top of TCP; needs a handshake before sending data |
| `nc` | Plain-text TCP connections; useless for TLS services |
| `ncat --ssl host port` | Netcat with TLS support (doesn't verify the certificate by default) |
| `openssl s_client -connect host:port` | Generic TLS client; shows certificate and session details |
| `-quiet` / `-ign_eof` | Disable interactive commands (`Q`, `R`, `k`, `K`) and don't close on EOF |
| `<` redirection or pipe | Send the password without copy-pasting |

{% include series-nav.html %}
