# Notes: What I Broke in SSH and How I Found the Cause

A lab exercise on SSH from week one of my learning plan. It covers three deliberately created failures: what I did, what the symptoms were, how I found the cause, and how I fixed it.

## Environment

- Server OS: Ubuntu 26.04
- OpenSSH version (`ssh -V`): OpenSSH_10.2p1 Ubuntu-2ubuntu3.6, OpenSSL 3.5.5 27 Jan 2026
- Client: MacOS Sequoia v15.7.7
- Fallback access in case of a mistake: UTM console

---

## Scenario A: Overly Open Permissions on `~/.ssh` on the Server

**What I did.** On the server, I ran `chmod 777 ~/.ssh`.

**Symptom.**

On the new connection attempt I receive an output:
user@203.0.113.10: Permission denied (publickey)

**How I looked for the cause.**

1. On the client: `ssh -v lab`. Right public key was offered, host is known but attempt ended with output: user@203.0.113.10: Permission denied (publickey)
2. On the server (via the fallback session): `sudo journalctl -u ssh -n 50`. Message:
   Authentication refused: bad ownership or modes for directory /home/user/.ssh

**Cause.** SSH refused to accept the key because of incorrect file permissions on the server

**Fix.** `chmod 700 ~/.ssh`. And after that login works correct

**Takeaway.** This taught me why the server is so strict about permissions.
