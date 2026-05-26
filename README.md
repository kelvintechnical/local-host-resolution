# Lab: Configuring Local Host Resolution with `/etc/hosts`

**Series:** linux-ops-mastery — RHCSA Networking
**Subjects covered:** `/etc/hosts` format (IP FQDN aliases), order of fields, duplicate name hazards, `getent hosts` vs `ping`, NSS (`/etc/nsswitch.conf`) **files** vs **dns** precedence, why local overrides exist, editing safely with copy-and-replace
**Career arcs covered:** RHCSA (name resolution triage appears constantly on EX200), RHCE (Ansible `lineinfile` / templates for cluster `/etc/hosts`), SRE (bootstrap before DNS exists in DR), DevOps (temporary service mesh discovery before Consul), AI/MLOps (multi-node training rings that hardcode peer names)
**Prerequisite:** Lab 36 or equivalent — you can `ping` an IP and know what a hostname is
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 backup + baseline · 2–3 single and multi-alias mappings · 4 prove NSS order · 5 duplicate / typo edge cases · 6 RHCSA capstone + cleanup

---

## Objective

Master **local static hostname mapping** the way a senior admin does: with backups, predictable ordering, and verification commands that do not lie. By the end of this lab you can add IPv4 and IPv6 entries to `/etc/hosts`, explain how those entries interact with DNS, and prove which source answered a given `getent` query.

The capstone is the RHCSA-realistic framing: *"Map the hostname `filer.lab.local` to `198.51.100.20`, ensure `getent hosts filer.lab.local` returns that address, and document the change in `/root/hosts-lab.txt` — then remove only your lab lines without disturbing vendor entries."*

> **Lab safety note:** `/etc/hosts` is global. A wrong entry for `repo.almalinux.org` can brick `dnf`. Keep edits at the **bottom** in a marked block and always `cp` a timestamped backup first.

---

## Concept: `/etc/hosts` Is the First Dictionary the Resolver Consults

Most userland programs do not implement DNS themselves. They call **`getaddrinfo()`**, which walks **Name Service Switch (NSS)** sources listed in `/etc/nsswitch.conf`. On a default RHEL 9 host, the `hosts:` line usually mentions **`files`** before **`dns`**, meaning `/etc/hosts` wins for any name listed there.

```
  Application (ping, ssh, curl)
           │
           ▼
    getent hosts NAME   ──►  glibc NSS
           │
           ├── files  ──►  /etc/hosts   (static table)
           │
           ├── dns    ──►  stub resolver / NetworkManager / upstream DNS
           │
           └── (others: myhostname, mdns_minimal, …)
```

Each non-comment line in `/etc/hosts` follows:

```
IP canonical_hostname [aliases ...]
```

> **Why this matters:** Clusters, kickstarts, and air-gapped installs frequently have **no DNS yet**. `/etc/hosts` is how you still `ssh node2`. Mistakes here look like “DNS is broken” when DNS was never consulted.

---

## 📜 Why `/etc/hosts` Exists — The Story

Before the ARPANET standardized the **Domain Name System** in the early 1980s, every participating machine kept a local table mapping human names to addresses. That table was literally a file shared by email and tape — the **`HOSTS.TXT`** monolith maintained by the Stanford Research Institute. As the network grew, downloading the whole universe of names every week stopped scaling.

DNS replaced the single file with a hierarchical, distributed database. Linux and BSD kept the **local** idea as `/etc/hosts` (path names vary historically, but the concept is universal): a tiny, administrator-controlled table consulted **first** for bootstrap paths, management networks, and emergency overrides.

The modern lesson is not nostalgia — it is **operational layering**. `/etc/hosts` answers instantly with zero UDP packets. DNS answers after cache TTLs, views, split-horizon forwarding, and TLS dependencies on correct names. SREs still add one line to `/etc/hosts` during a Sev-1 when the authoritative zone is wrong but a specific jump host IP is known-good.

> **The point of the story:** `/etc/hosts` is not a competitor to DNS — it is the **fast path** and the **override path**. Knowing when each runs is what separates triage from superstition.

---

## 👪 The Local Host Resolution Family — Who Lives There

### By file

| File | Role |
|---|---|
| `/etc/hosts` | Static IP ↔ name mappings |
| `/etc/nsswitch.conf` | Orders NSS sources (`files`, `dns`, …) |
| `/etc/resolv.conf` | Resolver configuration (often indirect on RHEL 9) |
| `/etc/host.conf` | Legacy glibc behavior (rarely edited today) |

### By command

| Command | What it proves |
|---|---|
| `getent hosts NAME` | The **actual** database answer glibc would use |
| `getent ahosts NAME` | May show multiple socktypes / orders |
| `ping NAME` | Implicitly resolves — good smoke test |
| `dig NAME` | Talks **DNS only** — bypasses `/etc/hosts` |

### By line shape

| Pattern | Example | Meaning |
|---|---|---|
| IPv4 + FQDN | `192.0.2.10 server.lab.local` | `server.lab.local` resolves to that IPv4 |
| IPv4 + aliases | `192.0.2.10 server.lab.local s1` | `s1` is extra shorthand |
| IPv6 | `2001:db8::1 v6.lab.local` | Same for IPv6 |
| Comment | `# lab block begin` | Ignored |

> **The point of the family tree:** If `getent` and `dig` disagree, that is **not** a mystery — they are querying different layers.

---

## 🔬 The Anatomy of a `/etc/hosts` Line — In One Diagram

```
192.0.2.20   filer.lab.local filer
└─────┬────┘ └──────┬────────┘ └─┬┘
      │             │            └ optional short alias (more may follow)
      │             └ canonical name (should match reverse DNS in real life)
      └ IPv4 address (v6 addresses can contain colons — keep fields aligned)

Common beginner bug:
127.0.0.1   myrealserver.example.com   # WRONG role for loopback
            └── loopback should name *this machine*, not remote peers
```

> **Reading rule:** The **first hostname** after the IP is the canonical name for that line; everything else on the line is an alias.

---

## 📚 `/etc/hosts` Reference Table

| Task | Command / action | Notes |
|---|---|---|
| Backup | `cp /etc/hosts /etc/hosts.bak.$(date +%F-%H%M%S)` | Always |
| Show tail | `tail -n +1 /etc/hosts` | Review before edit |
| Append block | `printf '\n# LAB\n192.0.2.1 a\n' >> /etc/hosts` | Quick, but prefer heredoc in real life |
| Verify | `getent hosts a` | Must return `192.0.2.1` |
| DNS-only check | `dig +short a @8.8.8.8` | Shows public DNS (may differ) |
| Remove block | Edit file deleting LAB section | Do **not** blindly `sed` vendor lines |

> **Rule one of hosts files:** When `getent` shows the wrong IP, check **`files` before `dns`** — your answer may already be in `/etc/hosts`.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Name resolution questions often reduce to “what does `getent hosts` print?” |
| **RHCE candidate** | You will template `/etc/hosts` across 200 nodes for a migration window. |
| **SRE / Platform** | Incident bridges use `/etc/hosts` to bypass broken internal DNS without touching app code. |
| **DevOps** | Ephemeral CI runners sometimes inject GitHub Enterprise hostnames before DNS propagates. |
| **AI / MLOps** | Distributed jobs reference `worker-0` … `worker-N`; hosts files bootstrap until CoreDNS is live. |

---

## 🔧 The 6 Tasks

> Six phases: **backup → map → verify → NSS → edge → capstone**.

---

### Task 1 — Backup `/etc/hosts` and capture baseline resolution

**Purpose:** Create a rollback file and record how your system currently resolves `localhost` and the machine’s own hostname.

```bash
sudo -i
cp -a /etc/hosts "/etc/hosts.bak.$(date +%F-%H%M%S)"
wc -l /etc/hosts
grep -E '^127\.' /etc/hosts || true

getent hosts localhost
hostname -f
getent hosts "$(hostname -f)" || echo "no hosts-file mapping for FQDN (normal)"
```

**Human-Readable Breakdown:** Timestamped copy preserves permissions. Count lines so you can sanity-check later edits. Show standard loopback lines. `getent hosts localhost` should return `127.0.0.1` and possibly `::1`.

**Reading it left to right:** `cp -a` preserves mode/owner. `getent` is the NSS-aware query tool — prefer it over `host` or `nslookup` when diagnosing `/etc/hosts`.

**The story:** The loopback stanza is sacred. This task makes sure you can **see** it before you append lab junk.

**Expected output:**

```text
45 /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
::1         localhost localhost.localdomain
127.0.0.1   localhost localhost.localdomain …
client1.lab.example.com client1
no hosts-file mapping for FQDN (normal)
```

**Switches**

| Token | Meaning |
|---|---|
| `cp -a` | Archive mode — preserve metadata |
| `getent hosts` | Query the `hosts` NSS database |
| `hostname -f` | FQDN if configured |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `getent: not found` | Extremely minimal container — install `glibc-common` / use full RHEL userspace |
| Permission denied on copy | Run as root |

---

### Task 2 — Add a single IPv4 mapping and verify

**Purpose:** Append one **lab-only** line mapping a fake FQDN to a documentation-prefix address you control.

```bash
printf '\n# --- LAB: local-host-resolution ---\n198.51.100.10   app.lab.local\n' >> /etc/hosts

tail -n 5 /etc/hosts
getent hosts app.lab.local
ping -c 1 app.lab.local || echo "ping may fail if nothing answers at 198.51.100.10"
```

**Human-Readable Breakdown:** The `198.51.100.0/24` TEST-NET-2 range is reserved for documentation — it should not collide with real routed Internet targets in examples. `getent` must return your IP even if ICMP fails.

**Reading it left to right:** Appending with a marked banner keeps your block grep-able. `ping` triggers resolution via NSS; failure at ICMP is independent of name resolution success.

**The story:** This is the smallest complete hosts edit: one IP, one canonical name, proof via `getent`.

**Expected output:**

```text
# --- LAB: local-host-resolution ---
198.51.100.10   app.lab.local
198.51.100.10   app.lab.local
PING app.lab.local (198.51.100.10) 56(84) bytes of data.
--- 198.51.100.10 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
ping may fail if nothing answers at 198.51.100.10
```

**Switches**

| Token | Meaning |
|---|---|
| `printf '...\n' >> file` | Append exact bytes |
| `ping -c 1` | Single probe |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `getent` returns nothing | Typo in name — check `tail` output |
| Wrong IP returned | Duplicate earlier line wins first match — search `grep -n 'app.lab' /etc/hosts` |

---

### Task 3 — Add aliases and an IPv6 parallel mapping

**Purpose:** Practice **multiple names per IP** and a **`2001:db8::/32` documentation IPv6** entry.

```bash
cat >> /etc/hosts <<'EOF'
198.51.100.20   db.lab.local database
2001:db8:ca2:9::1   v6.lab.local v6api
EOF

getent hosts db.lab.local
getent hosts database
getent ahosts v6.lab.local | head
```

**Human-Readable Breakdown:** Second field is canonical; `database` is an alias. IPv6 addresses belong in `/etc/hosts` like IPv4 — fields are whitespace-separated.

**Reading it left to right:** `getent ahosts` may print SOCK_STREAM vs SOCK_DGRAM duplicates — that is normal.

**The story:** Application connection strings often use short aliases (`database:5432`). Operators map those locally until DNS records exist.

**Expected output:**

```text
198.51.100.20   db.lab.local database
198.51.100.20   db.lab.local
2001:db8:ca2:9::1 STREAM v6.lab.local
2001:db8:ca2:9::1 DGRAM
2001:db8:ca2:9::1 RAW
```

**Switches**

| Token | Meaning |
|---|---|
| `cat >> file <<'EOF'` | Here-document append without variable expansion |
| `getent ahosts` | Address info including socktypes |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| IPv6 line ignored | Syntax error — confirm colons not accidentally commented |
| Only IPv4 returned for dual-stack name | Separate names per address are clearer than overloading one |

---

### Task 4 — Prove NSS order: `/etc/hosts` overrides public DNS for the same name

**Purpose:** Intentionally map a **real public DNS name** to a **loopback address** in the lab block to show precedence — then remove that dangerous line immediately after proof.

```bash
cp -a /etc/hosts /etc/hosts.before-override
grep -n 'lab\.example\.invalid' /etc/hosts || echo "safe: not present yet"

printf '\n127.0.0.1   lab.example.invalid\n' >> /etc/hosts
getent hosts lab.example.invalid

# undo override instantly:
grep -v 'lab.example.invalid' /etc/hosts > /tmp/hosts.clean && mv /tmp/hosts.clean /etc/hosts
getent hosts lab.example.invalid || echo "now unresolved via files (expected)"
```

**Human-Readable Breakdown:** We use a fake TLD **`.invalid`** reserved by RFC 2606 so we never accidentally hijack a real service. Still, the demonstration shows **`files` wins**.

**Reading it left to right:** `grep -v` rebuilds the file without the lab line — careful with this pattern on production files (race-safe editing uses `install`/`atomic` helpers). Here it is instructional.

**The story:** Junior admins blame BIND when `/etc/hosts` had a stale VIP from last year’s migration. This task trains the reflex: **`getent` first**.

**Expected output:**

```text
127.0.0.1   lab.example.invalid
now unresolved via files (expected)
```

**Switches**

| Token | Meaning |
|---|---|
| `.invalid` TLD | Guaranteed non-public — safe didactic override |
| `grep -v PATTERN` | Filter lines out |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv` lost permissions | Prefer `install -m 644 /tmp/hosts.clean /etc/hosts` in real life |
| Name still resolves | `systemd-resolved` cache — wait or `resolvectl flush-caches` |

---

### Task 5 — Edge case: duplicates, tabs vs spaces, and `dig` disagreement

**Purpose:** See what happens when the **same hostname appears twice** — first match behavior — and confirm `dig` still queries DNS servers, not `/etc/hosts`.

```bash
cat >> /etc/hosts <<'EOF'
198.51.100.30   dup.lab.local
203.0.113.30    dup.lab.local
EOF
getent hosts dup.lab.local

dig +short dup.lab.local @8.8.8.8 || echo "NXDOMAIN or timeout in lab is OK"
```

**Human-Readable Breakdown:** Many libc implementations return the **first** matching line for `getent hosts`. `dig` asks DNS directly — it will not show your `/etc/hosts` fiction.

**Reading it left to right:** Duplicate names are tech debt. Monitoring tools that use different APIs may disagree.

**The story:** This is the interview answer: “We standardized on **`getent`** for checks because `dig` bypasses NSS.”

**Expected output:**

```text
198.51.100.30   dup.lab.local
;; ANSWER SECTION may be empty
NXDOMAIN or timeout in lab is OK
```

**Switches**

| Token | Meaning |
|---|---|
| `dig +short` | Minimal answer if RR exists |
| `@8.8.8.8` | Force server |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Second line “wins” | Actually libc returns first — delete the stale line entirely |
| `dig` not installed | `dnf install bind-utils` |

---

### Task 6 — Capstone: `filer.lab.local` mapping + evidence file + surgical cleanup

**Task statement:** *"Append a marked lab block mapping `filer.lab.local` to `198.51.100.20`, verify with `getent hosts`, copy the block into `/root/hosts-lab.txt`, then remove **only** the lab block restoring original line count behavior."*

**Purpose:** Produce grader-friendly artifacts without touching vendor `localhost` lines.

```bash
sudo -i
TS=$(date +%F-%H%M%S)

cat >> /etc/hosts <<EOF
# --- LAB CAPSTONE $TS BEGIN ---
198.51.100.20   filer.lab.local filer
# --- LAB CAPSTONE $TS END ---
EOF

getent hosts filer.lab.local | tee /root/hosts-lab.txt
grep -n 'LAB CAPSTONE' /etc/hosts
```

**Cleanup**

```bash
BEGIN=$(grep -n 'LAB CAPSTONE' /etc/hosts | head -1 | cut -d: -f1)
END=$(grep -n 'LAB CAPSTONE' /etc/hosts | tail -1 | cut -d: -f1)
if [ -n "$BEGIN" ] && [ -n "$END" ]; then
  sed "${BEGIN},${END}d" /etc/hosts > /tmp/hosts.no-lab
  install -m 644 /tmp/hosts.no-lab /etc/hosts
fi
rm -f /tmp/hosts.no-lab /root/hosts-lab.txt

# remove earlier lab lines from Tasks 2–5 if you appended them:
sed -i '/# --- LAB: local-host-resolution ---/,/v6api/d' /etc/hosts 2>/dev/null || true
sed -i '/dup\.lab\.local/d' /etc/hosts

exit
```

**Expected output:**

```text
198.51.100.20   filer.lab.local filer
47:# --- LAB CAPSTONE 2026-05-26-101530 BEGIN ---
48:198.51.100.20   filer.lab.local filer
49:# --- LAB CAPSTONE 2026-05-26-101530 END ---
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `sed` deleted localhost | Restore from `/etc/hosts.bak.*` immediately |
| `getent` stale | `nscd` rare — restart if present |
| Cleanup removed real lines | Narrow `sed` addresses to marker lines only |

---

## 🔍 `/etc/hosts` Decision Guide

```
Name resolves wrong?
  │
  ├── `getent hosts NAME` shows unexpected IP
  │       ├── grep NAME /etc/hosts   → hit?
  │       │       └── yes → fix/remove that line (files wins)
  │       └── no → DNS path: resolvectl / nmcli / forwarders
  │
  ├── `dig` disagrees with `getent`
  │       └── expected — dig ignores hosts file
  │
  └── Need temporary override for one admin laptop?
          └── append single line + comment WHY + ticket ID
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Timestamped backup + baseline `getent hosts localhost`
- [ ] 02 Append `app.lab.local` → `198.51.100.10` and verify
- [ ] 03 Add aliases + IPv6 `2001:db8:ca2:9::1` mapping
- [ ] 04 Prove NSS override with `.invalid` name, then remove
- [ ] 05 Duplicate `dup.lab.local` lines + `dig` contrast
- [ ] 06 Capstone `filer.lab.local`, evidence file, marker-based cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Edited hosts without backup | Sev-1 | Restore `/etc/hosts.bak.*` |
| Put remote peers on `127.0.0.1` | Routing works, TLS SNI breaks | Use real IP |
| Trusted `ping` only | ICMP blocked but DNS OK | Use `getent` |
| Used `dig` to test hosts file | False negative | Use `getent hosts` |
| Duplicate names | Intermittent wrong IP | Delete stale line entirely |
| Tabs vs spaces | Rare parser pain | Keep single spaces between fields |
| Huge hosts file | Slow every lookup | Move to DNS or LDAP |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize: **`getent hosts NAME`** is the grading-friendly check.

**RHCE candidate**
- Show how you would **template** a hosts block with Ansible `blockinfile` and `marker`.

**SRE / Platform interview**
- Explain **split-horizon** vs `/etc/hosts` override — when each is ethical and safe.

**DevOps**
- Document that ephemeral containers should **inject** hosts at start, not require manual `vi`.

**AI / MLOps**
- For Ray/Slurm-style clusters, hosts files are the bootstrap; migrate to DNS once API server is up.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 36 — `nmcli` Network Config | Sets DNS that follows after hosts misses |
| Lab 38 — DNS Servers / `resolv.conf` | Downstream of NSS `dns` |
| Lab 39 — SSH Key Auth | `ssh app.lab.local` depends on resolution |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
