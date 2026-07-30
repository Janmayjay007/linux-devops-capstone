# Linux Server Onboarding & Incident Response

A hands-on Linux fundamentals project simulating a real junior DevOps task: provisioning a fresh Ubuntu server for a small dev team, then diagnosing and resolving a service incident on that same server.

## Overview

This project covers two phases that mirror a realistic engineering workflow:

1. **Setup & Hardening** — provisioning users, groups, and permissions for a small team, plus installing and enabling a web server
2. **Incident Response** — diagnosing why a "running" service was actually not serving any traffic, and fixing it

Along the way, an unplanned but genuinely useful third thread emerged: a disk-usage investigation that turned into a lesson on how `du` can misreport space when it crosses filesystem mount points — a real gotcha that trips up experienced engineers too.

**Environment:** Ubuntu (VM), Apple Silicon host

---

## Phase 1: Server Setup & Hardening
### Nginx running successfully
### Requirements

- Two users with different privilege levels: an admin (`devops_lead`) and a restricted developer (`junior_dev`)
- A shared group (`webteam`) with a shared directory accessible only to its members
- nginx installed, active, and enabled on boot

### What I did

**Created the users and set privilege levels:**

```bash
sudo adduser devops_lead
sudo usermod -aG sudo devops_lead      # admin privileges
sudo adduser junior_dev                # left out of sudo — restricted by design
```

Verified the split worked as intended by testing both accounts directly, rather than assuming group membership was enough:

```bash
$ su - junior_dev
$ sudo whoami
sudo: I'm sorry junior_dev. I'm afraid I can't do that
```

**Created the shared group and directory:**

```bash
sudo groupadd webteam
sudo usermod -aG webteam devops_lead
sudo usermod -aG webteam junior_dev

sudo mkdir -p /srv/webteam
sudo chown :webteam /srv/webteam
sudo chmod 770 /srv/webteam
```

Result — group has full access, everyone else has none:

```
drwxrwx--- 2 root webteam 4096 Jul 26 07:54 /srv/webteam
```

**Installed and enabled nginx:**

```bash
sudo apt install nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### A gotcha worth documenting

Running `systemctl enable nginx` *without* `sudo` failed — even though `devops_lead` is in the `sudo` group. Ubuntu's desktop build routed the request through a **polkit** authentication prompt instead of a normal permission check, and that failed with `Permission denied`. Group membership in `sudo` doesn't auto-elevate every command — `sudo` has to be explicitly invoked each time. Prefixing the command with `sudo` resolved it immediately.

### Unplanned detour: the disk was already at 99%

While recording a "baseline" `df -h` for later comparison, I noticed the root filesystem was already at 99% used with only 110M free — on what was supposed to be a fresh VM. Rather than ignore it, I investigated:

```bash
sudo du -sh /* 2>/dev/null | sort -rh | head -10
```
```
5.8G  /usr
5.1G  /snap
2.7G  /var
2.0G  /swap.img
```

`/snap` stood out. Checking `snap list --all` revealed old, disabled revisions of `mesa-2404`, `snapd`, and `thunderbird` sitting alongside their current versions — Snap keeps old revisions around indefinitely unless told otherwise, and `apt autoremove` doesn't touch them.

**Cleanup:**
```bash
snap list --all | awk '/disabled/{print $1, $3}' | while read snapname revision; do
  sudo snap remove "$snapname" --revision="$revision"
done
```
This removed the three stale revisions cleanly. However, `df -h` on the root filesystem **didn't change** afterward.

**Chasing that down:**
- Checked for deleted-but-still-open files holding disk space (`lsof +L1`) — ruled out, nothing significant found on real disk devices.
- Checked `mount | grep snap` — each snap package is its own mounted squashfs filesystem under `/snap`.
- Realized `du -sh /snap/*` had been walking *into* those separate mounts and summing their uncompressed apparent size, not their real compressed footprint on disk.
- Confirmed the accurate number by checking the actual snap storage location instead:
  ```bash
  sudo du -sh /var/lib/snapd/snaps
  # 1.4G (down from a reported 5.1G)
  ```

**Conclusion:** the cleanup was real and worthwhile, but the original "5.1G" figure was inflated by `du` crossing mount points — not an accurate picture of reclaimable space. The 12G disk being nearly full turned out to be mostly genuine usage (`/usr` + `/var` + real snap data + swap), not a single hidden culprit. Sometimes the correct finding in an investigation is "the disk is undersized for the workload," not "here's the one big file to delete" — and knowing the difference matters.

**Takeaway:** `du` can cross filesystem/mount boundaries and overcount. Use `du -x` to stay within a single filesystem when auditing disk usage that includes mount points like `/snap`.

---

## Phase 2: Incident Response

**Scenario:** A teammate reports the internal site is throwing an error.

### Investigation

```bash
curl http://localhost
```
```
curl: (7) Failed to connect to localhost port 80: Could not connect to server
```

Checked whether nginx itself had crashed:
```bash
systemctl status nginx
```
```
Active: active (running) since ...
Main PID: 75783 (nginx)
   ├─ master process
   └─ 4 worker processes
```

nginx was fully running — no crash, no restart loop. That ruled out the obvious explanation and meant the problem was something else. Checked whether anything was actually listening on port 80:

```bash
sudo ss -tlnp | grep :80
```
```
(no output)
```

Nothing was bound to port 80 at all, despite nginx being "active." Checked the enabled sites:

```bash
ls -l /etc/nginx/sites-enabled/
```
```
total 0
```

**Root cause:** `/etc/nginx/sites-enabled/` was empty. That directory is included directly by nginx's main config — it's not just a naming convention, it controls which server blocks nginx actually loads. With it empty, nginx starts successfully (an empty config include isn't a syntax error) but has no `server { listen 80; }` block to bind to any port. The process is alive; it just isn't serving anything.

### Fix

```bash
sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
sudo systemctl restart nginx
curl http://localhost   # returns the nginx welcome page — confirmed working
```

### Key lesson

`systemctl status` showing `active (running)` only confirms the **process** didn't die — it says nothing about whether the service is actually doing its job. A service can be fully "healthy" by that check while serving zero real traffic. This is exactly why production systems rely on **health checks** rather than process status alone to determine if something is actually working.

---

## Skills Demonstrated

- Linux user/group administration and the principle of least privilege
- File permission design for shared team access (`770`, group ownership)
- `systemd`/`systemctl` service management, including the enabled vs. active distinction
- Log-driven and network-state-driven incident diagnosis, rather than assumption-driven
- Disk usage investigation using `df`, `du`, `lsof`, and `mount` — including recognizing and correcting a flawed initial hypothesis
- Recognizing the gap between "process is running" and "service is functioning"

## What I'd Do Differently in Production

- Enable a real health-check endpoint rather than relying on `systemctl status` alone
- Configure `journald`/log rotation limits up front, since `/var/log/journal` was already one of the largest consumers of disk on this VM
- Use `du -x` by default when auditing disk usage on systems with Snap or other mounted-filesystem package managers
