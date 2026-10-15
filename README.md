# Zero Trust Server Access — Okta ASA on AWS EC2

Replacing static SSH keys with ephemeral identity-bound certificates using Okta Advanced Server Access (ASA) on AWS EC2. No shared credentials. Full session auditability.

> Implemented in production at Navat Technology

---

## The Problem with Static SSH Keys

| Static SSH Keys | Okta ASA (Zero Trust) |
|---|---|
| Keys never expire | Certificates expire after each session |
| Shared across team | Bound to individual identity |
| No audit trail | Full session recording |
| Manual rotation | Automatic lifecycle |
| One key = full access | Least-privilege per user |

---

## Architecture

```
Developer
    │
    ▼
Okta Identity (SSO login)
    │
    ▼
Okta ASA Gateway
    │  (issues ephemeral SSH certificate — valid for this session only)
    ▼
AWS EC2 Instance (ASA agent installed)
    │
    └── Audit log → Okta dashboard
```

---

## Setup Steps

### 1. Okta ASA Team Setup
- Create Okta ASA team at `app.scaleft.com`
- Add users and assign to a group (e.g., `devops-team`)

### 2. Create a Project
- In ASA dashboard → Projects → New Project
- Assign the `devops-team` group to the project

### 3. Install ASA Agent on EC2

```bash
# Add Okta ASA repo
curl -C - https://dist.scaleft.com/GPG-KEY-OktaPAM-2023 | gpg --dearmor | sudo tee /usr/share/keyrings/oktapam-2023-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/oktapam-2023-archive-keyring.gpg] https://dist.scaleft.com/repos/deb focal okta" | sudo tee /etc/apt/sources.list.d/oktapam.list

# Install agent
sudo apt update
sudo apt install scaleft-server-tools -y

# Enroll server into project
sudo sftd --enrollment-token <token-from-asa-dashboard>

# Start and enable
sudo systemctl enable sftd
sudo systemctl start sftd
sudo systemctl status sftd
```

### 4. Install ASA Client (Developer Machine)

```bash
# Mac
brew install scaleft/tap/sft

# Login
sft login

# List available servers
sft list-servers

# SSH into EC2 (certificate issued automatically)
sft ssh <server-name>
```

### 5. Configure Access Policies

In ASA dashboard → Project → Access → Configure:

```
Group: devops-team
Server access: Allow
Sudo access: Allow (for specific users only)
Session recording: Enabled
Certificate TTL: 1 hour
```

---

## Security Outcome

- **No static keys** — `.pem` files eliminated from the workflow
- **Ephemeral certs** — each session gets a new certificate, auto-expires
- **Identity-bound** — every SSH session tied to an Okta user identity
- **Least privilege** — sudo access granted per user, not per server
- **Full audit trail** — every session logged with user, timestamp, commands

---

## IAM Integration

ASA works alongside AWS IAM:

- ASA controls **who can SSH** into EC2
- IAM controls **what AWS resources** the EC2 instance can access
- Together: full Zero Trust posture for both server access and cloud API access

---

## Author

**Abishek S** — DevOps Engineer  
[LinkedIn](https://linkedin.com/in/abishek-iam) · [GitHub](https://github.com/abisheksug)
