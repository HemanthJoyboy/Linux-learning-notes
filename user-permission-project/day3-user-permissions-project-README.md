# Linux User & Group Permissions — Real Project Example 🔐

A simple, real-world scenario showing how to set up **least-privilege access** for two teams sharing one server: **DevOps** and **Development**.

---

## Scenario

A company runs a web app on an Ubuntu server. Two teams need access to the same code folder, but with different permission levels:

| Team | Users | Access Needed |
|---|---|---|
| **DevOps** | `user1`, `user2`, `user3` | Full control — read, write, execute (deploy, restart, manage) |
| **Development** | `user4`, `user5`, `user6` | Read + write only (write app code, no server management) |

One folder: `/opt/webapp`

---

## Step 1 — Create the groups

```bash
sudo groupadd devops
sudo groupadd development
```

## Step 2 — Create the users

```bash
sudo useradd -m -s /bin/bash user1
sudo useradd -m -s /bin/bash user2
sudo useradd -m -s /bin/bash user3
sudo useradd -m -s /bin/bash user4
sudo useradd -m -s /bin/bash user5
sudo useradd -m -s /bin/bash user6
```

Set a password for each:
```bash
sudo passwd user1   # repeat for user2 – user6
```

## Step 3 — Add each user to their team's group

```bash
sudo usermod -aG devops user1
sudo usermod -aG devops user2
sudo usermod -aG devops user3

sudo usermod -aG development user4
sudo usermod -aG development user5
sudo usermod -aG development user6
```

## Step 4 — Create the shared project folder

```bash
sudo mkdir -p /opt/webapp
sudo chown -R root:development /opt/webapp
sudo chmod -R 2775 /opt/webapp
```

- `root:development` → owner is root, group is the development team
- `2775` → owner: `rwx`, group: `rwx`, others: `r-x`
- Leading `2` = **setgid** — new files inside auto-inherit the `development` group

## Step 5 — Give DevOps full access too (via ACL)

```bash
sudo apt install acl -y
sudo setfacl -R -m g:devops:rwx /opt/webapp
sudo setfacl -R -d -m g:devops:rwx /opt/webapp
```

ACLs let a **second** group (`devops`) get full rights on the same folder, since a folder can only have one "main" group otherwise.

---

## Verify

```bash
groups user1              # should list: devops
groups user4               # should list: development
getfacl /opt/webapp        # confirm devops has rwx via ACL
```

Test as a developer:
```bash
sudo su - user4
touch /opt/webapp/app.py   # ✅ works
exit
```

Test as devops:
```bash
sudo su - user1
rm /opt/webapp/app.py      # ✅ works — devops has full control
exit
```

---

## Permission Cheat Sheet

| Symbol | Value | Meaning |
|---|---|---|
| `r` | 4 | read |
| `w` | 2 | write |
| `x` | 1 | execute |

| Combo | Value | Meaning |
|---|---|---|
| `rwx` | 7 | read + write + execute |
| `r-x` | 5 | read + execute |
| `r--` | 4 | read only |

---

## Key Commands Reference

| Command | What it does |
|---|---|
| `groupadd <name>` | Create a new group |
| `useradd -m -s /bin/bash <user>` | Create a user with a home dir + bash shell |
| `usermod -aG <group> <user>` | Add a user to a group (append, safe) |
| `chown -R owner:group <path>` | Set owner and group recursively |
| `chmod -R <number> <path>` | Set read/write/execute permissions recursively |
| `setfacl -R -m g:<group>:rwx <path>` | Grant a second group extra access via ACL |
| `getfacl <path>` | View all ACL rules on a folder |
| `groups <user>` | Check which groups a user belongs to |

---

## Why This Matters (Least Privilege Principle)

- ✅ Developers can write code, but can't restart services or delete server configs.
- ✅ DevOps can manage everything, but code changes are still traceable to the dev team's group.
- ✅ No one gets more access than their role needs — limiting damage if an account is compromised.
- ⚠️ Never `chmod 777` a project folder — it removes all access boundaries.

---
