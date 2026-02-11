---
title: UFW Practical Guide: Adding and Deleting Firewall Policies
---

# UFW Practical Guide: Adding and Deleting Firewall Policies

UFW (Uncomplicated Firewall) is a high-level frontend for `iptables`/`nftables` designed to simplify firewall management while preserving deterministic rule control. This article focuses specifically on **policy creation (add)** and **policy removal (delete)** in production environments.

---

# 1. Conceptual Model

Before manipulating rules, understand UFW’s evaluation logic:

* Rules are processed **top-to-bottom**
* **First match wins**
* Default policies apply only if no rule matches

Therefore, rule order is critical.

---

# 2. Core Terminology

| Term         | Meaning                                |
| ------------ | -------------------------------------- |
| `in`         | Incoming traffic to local host         |
| `out`        | Outgoing traffic from local host       |
| `route`      | Forwarded traffic (gateway mode)       |
| `on <iface>` | Restrict to specific network interface |
| `from`       | Source IP                              |
| `to`         | Destination IP                         |
| `port`       | Destination port                       |
| `proto`      | Protocol (tcp/udp)                     |

---

# 3. Adding Policies

## 3.1 Basic Allow Rule

Allow SSH:

```bash
sudo ufw allow 22/tcp
```

Equivalent explicit form:

```bash
sudo ufw allow in to any port 22 proto tcp
```

---

## 3.2 Allow from Specific IP

Allow SSH only from LAN:

```bash
sudo ufw allow from 192.168.0.0/16 to any port 22 proto tcp
```

---

## 3.3 Deny Rule

Block a port globally:

```bash
sudo ufw deny 10801/tcp
```

---

## 3.4 Interface-Specific Rule

Allow only via WireGuard:

```bash
sudo ufw allow in on wg0 to any port 83 proto tcp
```

---

## 3.5 Peer-Specific Deny (WireGuard Example)

Block client `10.0.0.3` from accessing local port `1728`:

```bash
sudo ufw insert 1 deny in on wg0 from 10.0.0.3 to any port 1728 proto tcp
```

Note the use of `insert 1` to ensure high priority.

---

## 3.6 Forwarded Traffic (Gateway Mode)

Allow routing from wg0 to LAN bridge:

```bash
sudo ufw route allow in on wg0 out on br0
```

Block specific routed service:

```bash
sudo ufw route deny in on wg0 from 10.0.0.3 to 192.168.1.10 port 80 proto tcp
```

---

# 4. Deleting Policies

UFW supports two deletion methods.

---

## 4.1 Delete by Exact Rule

If rule was added as:

```bash
sudo ufw allow 83/tcp
```

Remove using identical syntax:

```bash
sudo ufw delete allow 83/tcp
```

This requires exact matching parameters.

---

## 4.2 Delete by Rule Number (Preferred)

List rules:

```bash
sudo ufw status numbered
```

Example output:

```
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 1728/tcp                   DENY  IN    10.0.0.3 on wg0
```

Delete rule 2:

```bash
sudo ufw delete 2
```

Advantages:

* No need to retype full rule
* Avoids syntax mismatch
* Safer in complex environments

---

# 5. Modifying Rules

UFW does not support in-place editing.

Workflow:

1. Delete old rule
2. Re-add corrected rule

Example:

```bash
sudo ufw delete 3
sudo ufw insert 1 deny in on wg0 from 10.0.0.3 to any port 1728 proto tcp
```

---

# 6. Managing Rule Priority

### Append (default)

```bash
sudo ufw allow 83/tcp
```

Adds rule at bottom.

### Insert at specific position

```bash
sudo ufw insert 1 deny ...
```

Essential when overriding blanket allow rules.

---

# 7. Default Policies

View current policy:

```bash
sudo ufw status verbose
```

Set defaults:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

These apply only if no rule matches.

---

# 8. Resetting UFW

If configuration becomes inconsistent:

```bash
sudo ufw reset
```

This removes all rules and disables UFW.

Then re-enable:

```bash
sudo ufw enable
```

---

# 9. Production Best Practices

### 9.1 Avoid Blanket Interface Allow

Avoid:

```
ALLOW IN on wg0 from Anywhere
```

Instead:

* Default deny
* Explicit allow per service
* Specific deny exceptions at top

---

### 9.2 Keep Rule Set Minimal

Each rule increases evaluation complexity and cognitive load.

Prefer:

* Subnet-based grouping
* Interface-based segmentation
* Explicit service mapping

---

### 9.3 Separate Concerns

Do not mix:

* Raw `iptables` rules
* UFW-managed rules

Choose one firewall authority to avoid unpredictable order interactions.

---

# 10. Diagnostic Commands

Essential inspection commands:

```bash
sudo ufw status verbose
sudo ufw status numbered
sudo iptables -L -n -v --line-numbers
sudo nft list ruleset
```

These help identify:

* Rule order issues
* Backend type (iptables vs nftables)
* Packet counters

---

# 11. Mental Model for Secure Configuration

Recommended structure:

```
Default deny incoming
↓
Allow required services
↓
Insert specific deny rules above allows
↓
Test with tcpdump
```

Firewall design should be **explicit, minimal, and ordered**.

---

# Conclusion

UFW is deterministic and reliable when rule ordering is respected.
Most unexpected behavior results from:

* Blanket allow rules
* Incorrect rule priority
* Mixing firewall frameworks

Control priority deliberately using `insert`, delete rules precisely using numbered output, and maintain a minimal rule surface.

A firewall is not merely a filter — it is a policy engine. Design it accordingly.
