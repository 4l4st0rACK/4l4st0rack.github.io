---
title: "OverTheWire Bandit: Level 20 → 21 Walkthrough"
date: 2026-10-19 20:54:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, netcat, suid, tcp, bash]
description: OverTheWire Bandit level 20 to 21 walkthrough, setting up a netcat server to feed a SUID binary, using two terminals or a single background command.
ref: bandit-20-21
serie: bandit
nivel: 21
---

Until now we've used `nc` as a **client**: we connected to a service that was already listening. In this level we flip it around: we're going to set up our own **server** with `nc`, and a system binary will connect to us. It's the first time we use netcat in listen mode, something you'll do constantly when catching reverse shells in a pentest.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit20@bandit.labs.overthewire.org -p 2220
bandit20@bandit:~$ export TERM=xterm
```

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit21.html) says there's a SUID binary, `suconnect`, that:

1. Connects to a port on `localhost` that you give it on the command line.
2. Reads a line of text from that connection.
3. Compares it to the current level's password (bandit20).
4. If it's correct, it **sends back the next level's password** (bandit21).

```console
bandit20@bandit:~$ ls
suconnect
bandit20@bandit:~$ ./suconnect
Usage: ./suconnect <portnumber>
This program will connect to the given port on localhost using TCP. If it receives the correct password from the other side, the next password is transmitted back.
```

The important detail: `suconnect` **connects** to a port, it doesn't listen. It's a client. So someone has to be listening on that port and, when `suconnect` connects, send it bandit20's password. We set up that "someone" ourselves with `nc`.

## The solution: two terminals

### Terminal 1: set up the server with netcat

I open a **second** SSH session as bandit20 (another terminal) and set `nc` to listen on any port above 1024, for example 4444:

```console
bandit20@bandit:~$ nc -lnvp 4444
listening on [any] 4444 ...
```

Breakdown of the options:

| Option | What it does |
| --- | --- |
| `-l` | *Listen* mode: instead of connecting, it waits for connections (server) |
| `-n` | Don't resolve DNS; work with IPs directly (faster) |
| `-v` | *Verbose*: shows connection info when someone connects |
| `-p 4444` | The port to listen on |

> Why 4444 and not a lower one? Ports below 1024 are reserved and only `root` can open them. As a normal user you have to pick a high one. 4444 is a very common convention, but any free port between 1025 and 65535 works.
{: .prompt-info }

`nc` just hangs there, waiting. That's normal: it's listening.

### Terminal 2: launch suconnect against that port

In the **first** terminal I run the binary, telling it to connect to the port where `nc` is listening:

```console
bandit20@bandit:~$ ./suconnect 4444
```

At that moment, in the `nc` terminal you'll see a connection come in (thanks to `-v`). Now `suconnect` is waiting to receive a line of text.

### Back in Terminal 1: send the password

In the `nc` terminal, I type (or paste) bandit20's password and press Enter. `nc` sends it to `suconnect` over the connection, it compares it, and since it's correct **it responds with bandit21's password**, which appears in the same `nc` terminal:

```console
bandit20@bandit:~$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [127.0.0.1] from [127.0.0.1] ...
[REDACTED]        <- here I paste bandit20's password and press Enter
[REDACTED]        <- suconnect responds with bandit21's
```

And in the `suconnect` terminal you'll see a `Read: ...` and `Password matches, sending next password`.

## Other ways to do it

### 1. Without two terminals: everything in one command

Setting up two SSH sessions is a bit clunky. We can do it all in one terminal by leaving `nc` in the **background**:

```console
bandit20@bandit:~$ nc -lnvp 4444 < /etc/bandit_pass/bandit20 &
bandit20@bandit:~$ ./suconnect 4444
```

What happens here:

- `< /etc/bandit_pass/bandit20` feeds `nc` the password directly through standard input, so there's no need to paste it by hand. As soon as someone connects, `nc` sends it.
- The trailing `&` leaves `nc` running in the **background** and gives us the prompt back so we can launch `suconnect` right after.

When you run `suconnect 4444`, it connects to the `nc` running in the background, that sends it the password, and the reply (bandit21's) appears on screen.

### 2. A named pipe (FIFO)

A more "elegant" variant using a **named pipe**, in case the server needed a back-and-forth conversation:

```console
bandit20@bandit:~$ mkfifo /tmp/f
bandit20@bandit:~$ nc -lp 4444 < /etc/bandit_pass/bandit20 &
bandit20@bandit:~$ ./suconnect 4444
```

You don't need it for this level (the communication is one-way), but `mkfifo` is a pattern that shows up a lot when building interactive reverse shells, so it's worth knowing.

## Moving to the next level

```console
bandit20@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit21@bandit.labs.overthewire.org -p 2220
```

> The next level changes topic: **cron jobs** come into play, scheduled tasks the system runs automatically at set intervals.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `nc -lnvp 4444` | Set netcat to listen (server) on port 4444 |
| `-l` / `-p` | Listen mode / port |
| `-n` / `-v` | No DNS resolution / verbose mode |
| `./suconnect <port>` | SUID client that connects and waits for the password |
| `nc ... < file &` | Send a file over the connection and leave the process in the background |
| `&` | Run a command in the background |
| Ports > 1024 | The ones a non-root user can open |
| `mkfifo` | Create a named pipe (two-way communication) |

{% include series-nav.html %}
