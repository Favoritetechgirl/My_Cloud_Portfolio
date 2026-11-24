# Week 01: Shared Server Setup

In this task I set up a small Linux server that a team can share. I did everything by hand, and this README lists every step so anyone can repeat it on a fresh Ubuntu 26.04 VM.

## What I built

1. Two groups (`web` and `ops`) and three users (`dev1`, `dev2`, `op1`)
2. A shared website folder at `/srv/site`
3. A special user, `svc-site`, that runs the website
4. A service called `site.service` that serves the site on port 8081
5. A sudo rule that lets the `ops` group restart and check the service, and nothing else

## Before you start

- A fresh Ubuntu 24.04 VM
- A user that can run `sudo`
- Run all commands from the root of the repo

**Two small rules I followed:** I used lowercase for all names.
---

## Step 1: Create the groups and users

```bash
sudo groupadd web
sudo groupadd ops
sudo useradd -m -s /bin/bash -G web dev1
sudo useradd -m -s /bin/bash -G web dev2
sudo useradd -m -s /bin/bash -G web,ops op1
```

**What this does:** Groups let me give access to a whole team at once. `dev1` and `dev2` are in `web`. `op1` is in both `web` and `ops`.

**Check it worked:**

```bash
id dev1
id dev2
id op1
```

You should see `web` in the groups for all three, and `ops` for `op1` only.

**Undo it:**

```bash
sudo userdel -r dev1
sudo userdel -r dev2
sudo userdel -r op1
sudo groupdel web
sudo groupdel ops
```

---

## Step 2: Create the shared folder

```bash
sudo mkdir -p /srv/site
sudo chown root:web /srv/site
sudo chmod 2775 /srv/site
echo '<h1>Hello from the shared site</h1>' | sudo tee /srv/site/index.html
sudo chown root:web /srv/site/index.html
sudo chmod 664 /srv/site/index.html
```

**What this does:**

- `mkdir -p` creates the folder.
- `chown root:web` makes `root` the owner and `web` the group.
- `chmod 2775` sets the permissions:
  - `2` means new files inside get the `web` group automatically
  - first `7` = the owner can read, write and enter
  - second `7` = the `web` group can read, write and enter
  - `5` = everyone else can only read and enter
- The last two lines create a simple home page that the `web` group can edit.

**Check it worked:**

```bash
ls -ld /srv/site
```

You should see `drwxrwsr-x root web`. The `s` shows the group setting is on.

**Undo it:**

```bash
sudo rm -rf /srv/site
```

---

## Step 3: Create the service user

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin svc-site
```

**What this does:** This makes a user that exists only to run the website. It has no home folder and nobody can log in as it. Running the site as its own limited user is safer than running it as root.

**Check it worked:**

```bash
getent passwd svc-site
```

The line should end with `/usr/sbin/nologin`.

**Undo it:**

```bash
sudo userdel svc-site
```

---

## Step 4: Create the website service

The service is described in a file called `site.service`:

```ini
[Unit]
Description=Static site on port 8081
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=svc-site
Group=svc-site
WorkingDirectory=/srv/site
ExecStart=/usr/bin/python3 -m http.server 8081 --bind 0.0.0.0
Restart=on-failure
RestartSec=3
StandardOutput=journal
StandardError=journal
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Install and start it:

```bash
sudo cp week01/site.service /etc/systemd/system/site.service
sudo systemctl daemon-reload
sudo systemctl enable --now site.service
```

**What the important lines mean:**

- `User=svc-site`: runs as the user from Step 3
- `ExecStart=... 8081`: starts Python's simple web server on port 8081
- `Restart=on-failure`: if it crashes, it restarts after 3 seconds
- `StandardOutput=journal`: logs go to the system journal
- `WantedBy=multi-user.target` plus `enable`: starts the service when the server boots
- `daemon-reload`: tells systemd to read the new file
- `enable --now`: starts it right now and on every boot

**Check it worked:**

```bash
systemctl status site.service --no-pager
curl localhost:8081
```

The status should say `active (running)` and `curl` should print the home page.

If something goes wrong, read the logs:

```bash
journalctl -u site.service -n 30 --no-pager
```

**Undo it:**

```bash
sudo systemctl disable --now site.service
sudo rm /etc/systemd/system/site.service
sudo systemctl daemon-reload
```

---

## Step 5: Let the ops group manage the service

The rule is saved in `ops-site.sudoers`:

```
%ops ALL=(root) NOPASSWD: /usr/bin/systemctl restart site.service, /usr/bin/systemctl status site.service
```

Check it for mistakes first, then install it:

```bash
sudo visudo -cf week01/ops-site.sudoers
sudo install -m 440 -o root -g root week01/ops-site.sudoers /etc/sudoers.d/ops-site
sudo visudo -c
```

**What the rule means:** Anyone in the `ops` group can run those two exact commands as root without typing a password. They cannot run anything else, not even `systemctl stop site.service`.

**Things to be careful about:**

- Always check the file with `visudo` before installing it. A mistake in sudo files can lock you out.
- The installed file is called `ops-site` with no dot. Sudo ignores files in `/etc/sudoers.d/` that have a dot in the name.

**Check it worked:**

```bash
sudo -u op1 sudo -n systemctl status site.service     # should work
sudo -u op1 sudo -n systemctl restart site.service    # should work
sudo -u op1 sudo -n systemctl stop site.service       # should be refused
sudo -u dev1 sudo -n systemctl restart site.service   # should be refused
```

**Undo it:**

```bash
sudo rm /etc/sudoers.d/ops-site
```

---

## Start over completely

To remove everything, run these in order:

```bash
sudo systemctl disable --now site.service
sudo rm -f /etc/systemd/system/site.service
sudo systemctl daemon-reload
sudo rm -f /etc/sudoers.d/ops-site
sudo userdel -r dev1
sudo userdel -r dev2
sudo userdel -r op1
sudo userdel svc-site
sudo groupdel web
sudo groupdel ops
sudo rm -rf /srv/site
```

The fastest way is to restore the VM from its `clean` snapshot.

## Final check

When all the steps are done, I run:

```bash
curl localhost:8081
```

Every check should say `PASS`, and `curl` should show the home page.
