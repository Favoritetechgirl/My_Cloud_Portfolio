# Week 01: Shared Server Setup

In this task I set up a small Linux server that a team can share. I did everything by hand, and this README lists every step so anyone can repeat it on a fresh Ubuntu 24.04 VM.

## What I built

1. Two groups (`web` and `ops`) and three users (`dev1`, `dev2`, `op1`)
2. A shared website folder at `/srv/site`
3. A special user, `svc-site`, that runs the website
4. A service called `site.service` that serves the site on port 8081
5. A sudo rule that lets the `ops` group restart and check the service, and nothing else

## Before you start

- A fresh Ubuntu 24.04 VM
- A user that can run `sudo`

**Two small rules I followed:** I used lowercase for all names.


