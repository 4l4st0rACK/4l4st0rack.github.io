---
title: "OverTheWire Bandit: Level 14 → 15 Walkthrough"
date: 2026-10-18 22:40:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, netcat, networking, ports, localhost]
description: OverTheWire Bandit level 14 to 15 walkthrough. The first networking level - send the current password to a service on port 30000 of localhost with nc, and learn what ports and loopback are.
ref: bandit-14-15
serie: bandit
nivel: 15
---

Up to the [previous level](/en/posts/bandit-level-13-14/) everything happened inside the filesystem: searching, filtering, decoding, decompressing. From here on, Bandit starts bringing **networking** into the picture. In this level there's a service listening on a port of the machine itself, and to get the password we have to **talk to it**. The tool for that is `nc` (netcat), the "Swiss Army knife" of network connections.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 14

```console
$ sshpass -p '[REDACTED]' ssh bandit14@bandit.labs.overthewire.org -p 2220
bandit14@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 15 page](https://overthewire.org/wargames/bandit/bandit15.html), the password for the next level can be retrieved by **submitting the password of the current level to port 30000 on localhost**.

```console
bandit14@bandit:~$ ls
bandit14@bandit:~$
```

The home directory is empty: no files to search this time. Everything goes through the network.

## Concepts: localhost and ports

Two ideas we'll need from now on:

- **`localhost`** is the name a machine uses to refer to **itself**. It points to the address `127.0.0.1`, the *loopback* interface: traffic sent there never leaves for any network, it stays inside the machine. Since we're connected to the Bandit server over SSH, `localhost` is that server.
- A **port** is a number (from 0 to 65535) that identifies which service a connection is meant for within a machine. A single IP address can host many services at once, each one "listening" on its own port: SSH usually uses 22 (in Bandit, 2220 from outside), a web server 80 or 443... Here someone has left a service listening on **30000**.

## The solution: nc

First we need the current password. We know it because we just used it to log in, but it's also in the file we saw in the previous level:

```console
bandit14@bandit:~$ cat /etc/bandit_pass/bandit14
[REDACTED]
```

Now we open a connection to the service:

```console
bandit14@bandit:~$ nc localhost 30000
[REDACTED]
```

`nc localhost 30000` opens a TCP connection to port 30000 on the machine itself and waits. From there, `nc` works like a pipe: **whatever we type is sent to the service, and whatever the service replies shows up on screen**. We paste `bandit14`'s password, press Enter, the service checks that it's correct and replies with `bandit15`'s password.

It's a very direct way to understand how many network protocols work under the hood: in the end, a service is just a program that reads whatever arrives on a port and answers.

## Other ways to do it

### 1. No copy and paste: redirection or a pipe

```console
bandit14@bandit:~$ nc localhost 30000 < /etc/bandit_pass/bandit14
```

Instead of typing the password by hand, we feed the file to `nc` through standard input with `<`, the same redirection we used with `tr` in [level 11 → 12](/en/posts/bandit-level-11-12/). `nc` sends the file's contents to the service and prints the reply. A pipe works too:

```console
bandit14@bandit:~$ cat /etc/bandit_pass/bandit14 | nc localhost 30000
```

Both avoid the usual pasting mistakes (an extra space, a missing character) and can also be used inside scripts.

### 2. See which ports are listening with `ss`

```console
bandit14@bandit:~$ ss -tln
```

Before connecting to a service, it's useful to know what's listening on the machine. `ss` shows connections and open ports:

| Option | What it does |
|---|---|
| `-t` | TCP connections only |
| `-l` | Only listening ports |
| `-n` | Shows ports as numbers instead of translating them to service names |

In the list it returns you should find `30000`, along with other ports used by later Bandit levels. In pentesting, enumerating open ports is almost always the first step. `ss` is the modern replacement for `netstat`, which does the same with `netstat -tln`.

### 3. Without nc: bash's `/dev/tcp` trick

```console
bandit14@bandit:~$ exec 3<>/dev/tcp/localhost/30000
bandit14@bandit:~$ cat /etc/bandit_pass/bandit14 >&3
bandit14@bandit:~$ cat <&3
```

Bash has a little-known built-in feature: if you redirect to a path of the form `/dev/tcp/host/port`, it opens a TCP connection. That file doesn't exist on disk; bash interprets it itself.

- `exec 3<>/dev/tcp/localhost/30000` opens the connection and attaches it to file descriptor `3`, for reading and writing (`<>`). We already know descriptors 0, 1 and 2 from [level 6 → 7](/en/posts/bandit-level-6-7/); 3 is the first free one.
- `cat ... >&3` sends the password over that connection.
- `cat <&3` reads the service's reply.

It's very handy on machines where `nc` isn't installed, which happens a lot in real environments.

## Moving on to the next level

```console
bandit14@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit15@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| `localhost` / `127.0.0.1` | The machine itself (loopback interface) |
| Port | A number identifying which service a connection is for |
| `nc host port` | Open a TCP connection and talk to the service |
| `nc host port < file` | Send a file's contents to the service |
| `ss -tln` | List listening TCP ports |
| `/dev/tcp/host/port` | A TCP connection from bash, no external tools |
| `exec 3<>` | Open a file descriptor for reading and writing |

{% include series-nav.html %}
