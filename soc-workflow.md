# 🔐SOC Workflow: Multiple Failed Login Attempts

**🧠 Simulated scenario** — This is a portfolio project showing how I would investigate and respond
to a brute force alert as a SOC analyst. All usernames, IPs, and timestamps are fictional
but based on realistic SOC patterns.

---

## 📌 Alert

| Field | Detail |
|---|---|
| Alert name | Multiple Failed Login Attempts |
| Severity | Medium |
| Source | Windows Security Event Log via Microsoft Sentinel |
| Event ID | 4625 (Failed logon) |
| Trigger | 5+ failed logins within 10 minutes from the same IP |
| Time fired | 2025-03-12 at 02:17 UTC |

Sentinel fired this alert at 02:17 UTC on a Wednesday night. The first thing I noticed
was the time — 2 AM is outside normal business hours for this account, which immediately
raised suspicion.

---

## 🔍 Investigation

### What I looked at first

**1. Which account was targeted?**
The failed attempts were against `jsmith@contoso.com` — a standard employee account
with no admin privileges. Not a service account, which ruled out automated tooling on our end.

**2. Where did the attempts come from?**
All 14 attempts came from a single external IP: `185.220.101.47`. I ran a quick geolocation
check — the IP resolved to Romania. Our user jsmith is based in Houston, TX. That mismatch
alone was enough to treat this seriously.

**3. How many attempts and over what time?**
14 failed logins between 02:11 and 02:17 UTC — roughly one attempt every 25 seconds.
That rhythm is consistent with an automated tool, not a person mistyping their password.

**4. Did any attempt succeed?**
Yes. At 02:19 UTC — two minutes after the last failure — a successful login was recorded
(Event ID 4624) from the same IP. This changed the severity from Medium to High immediately.

### My decision at this point

This is not a false positive. The combination of:
- External IP from an unexpected country
- Automated attempt pattern (every ~25 seconds)
- Successful login following the failures
- Login time outside business hours

...confirmed this as a credential stuffing attack. I moved straight to containment.

---

## 🚨 Response

### Step 1 — Contain (immediate)
Disabled the `jsmith` account in Azure Active Directory to cut off the attacker's access.
Blocked IP `185.220.101.47` at the firewall level.

### Step 2 — Assess the damage
Reviewed audit logs for the 2-minute window the attacker had access (02:19–02:21 UTC).
No files were opened, no emails sent, no privilege changes made. The attacker appears
to have gotten in but not done anything yet — possibly interrupted by the account lockout.

### Step 3 — Notify
Contacted jsmith via phone (not email, since the email account may be compromised).
Confirmed he was asleep and had not attempted any logins. Account compromise confirmed.

### Step 4 — Recover
Reset jsmith's credentials. Enforced MFA enrollment before re-enabling the account.
Checked whether the same password was used on any other internal systems (it was —
also reset those).

### Step 5 — Document and escalate
Filed full incident report in the ticketing system. Flagged to the senior analyst given
the confirmed successful login. No escalation to IR team needed given no data was accessed.

---

## 📝 Closure

At 03:45 UTC, approximately 1.5 hours after the alert fired, the incident was closed.

**Summary:** A credential stuffing attack against jsmith@contoso.com succeeded in
gaining initial access for approximately 2 minutes before being contained. No data
was exfiltrated. The account has been secured with a new password and MFA enforced.

**Classification:** True positive — confirmed external attack. Severity upgraded to High.

---

## 🔧 What I would do differently / improvements

- The alert threshold of 5 failed attempts in 10 minutes caught this, but the attacker
  still got in. A lower threshold (3 attempts) or an automatic account lock after failures
  would have prevented the successful login entirely.
- MFA was not enabled on this account before the incident. If it had been, the attacker
  would have been stopped even after getting the correct password.
- Adding IP geolocation as an automatic enrichment to the alert would speed up triage —
  I had to check that manually.

---

## Note on this project

I don't have access to a live SIEM for this exercise. The investigation steps, log data,
and decisions above are simulated based on how a real analyst would work through this
alert using Microsoft Sentinel and Windows Event Logs. The thought process and response
order reflect standard SOC procedures.
