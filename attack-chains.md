# Attack Chains

## Chain 1: RDP Brute Force

### Overview
This chain simulates a credential brute-force attack against a Remote Desktop 
Protocol (RDP) service exposed on a lab host, followed by detection of both the 
attempt and the resulting compromise via Wazuh. This maps to 
**MITRE ATT&CK T1110 (Brute Force)** and, where successful, **T1078 (Valid 
Accounts)** for the subsequent access.

**Attacker:** Kali Linux (`10.10.5.12`)
**Target:** Domain-joined host with RDP enabled (`10.10.5.10` DC01 or 
`10.10.5.11` Windows 10 client, depending on test run)
**Detection:** Wazuh Manager (`10.10.5.20`)

---

### Step 1 — Reconnaissance

A host discovery scan was run from Kali against the full lab subnet to enumerate 
live hosts before selecting a target using nmap. We can see the hosts that are 
up on the network:

\`\`\`bash
nmap -sn 10.10.5.0/24
\`\`\`

A targeted scan to initiate service discovery with the 
-sV flag on the domain controller.
This is done to ensure port 3389 is open for brute forcing.
You could also scan just port 3389:

\`\`\`bash
nmap -sV 10.10.5.10 or nmap -p 3389 10.10.5.10
\`\`\`

![nmap_1](images/nmap_1.png)

Another scan was initiated on the client. This information can be used to 
demonstrate the importance of secure passwords (the client has a less 
secure password than the server):

\`\`\`bash
nmap -sV 10.10.5.11
\`\`\`

![nmap_2](images/nmap_2.png)

---

### Step 2 — Brute force

Hydra was used to attempt authentication against the target's RDP service using 
the `rockyou.txt` wordlist against a known domain username. Rockyou can be un-
packed from the wordlists directory on Kali by default:

\`\`\`bash
hydra -l dante -P /usr/share/wordlists/rockyou.txt rdp://10.10.5.11 -t 1 -V -I
\`\`\`

**Notes from execution:**
- The target account's password had been intentionally weakened for this test 
  (Default Domain Policy password complexity temporarily disabled, password set 
  to a value present in the `rockyou.txt` wordlist) to ensure the brute force 
  would succeed within a reasonable number of attempts.
- Domain authentication required specifying the domain explicitly during manual 
  connection testing (`xfreerdp ... /d:cyber.local`), since RDP clients default 
  to attempting local machine authentication rather than domain authentication 
  when no domain is specified.
- The target domain account required explicit membership in the local **Remote 
  Desktop Users** group on the client to permit RDP logon — domain users do not 
  receive this access by default.

Hydra successfully identified the correct credentials after a small number of 
attempts.

![hydra](images/hydra.png)

This password was then used to connect to the client remotely with xfreerdp:

![xfreerdp](images/xfreerdp.png)

---

### Step 3 — Detection in Wazuh

The attack was visible in the Wazuh dashboard (**Threat Hunting → Events**) in 
two distinct stages:

**Failed authentication attempts:**
Each incorrect password guess generated a Windows Security Event (Logon Failure), 
surfaced by Wazuh as seen here:

![wazuh_1](images/wazuh_1.png)

The same attack was performed on the server, which has a more secure password,
generating even more logs:

![wazuh_2](images/wazuh_2.png)

**Successful authentication:**
Immediately following the failed attempts, a successful logon event appeared, 
confirming the compromise:

![wazuh_3](images/wazuh_3.png)

This failure-then-success pattern — many authentication failures from a single 
source IP followed immediately by a successful logon from that same source — is 
a textbook brute-force compromise signature, and was detected using Wazuh's 
default ruleset with no custom rule authoring required.

---

### Observations
- Full event detail (source IP, logon type, target account) was present in the 
  underlying event data but not shown by default in Wazuh's summary table view — 
  it required expanding the individual event's full document/JSON view to surface.
- RDP's use of TLS for the connection meant credentials were never visible in 
  plaintext during a parallel Wireshark capture of the attack traffic; detection 
  relied entirely on host-based logon telemetry (via the Wazuh agent) rather than 
  network-layer payload inspection.

  ---

## Chain 2: Privilege Escalation via Domain Admin Account Creation

### Overview
This chain simulates a post-compromise persistence/privilege escalation action, 
where an attacker who has already obtained valid domain credentials (via the RDP 
brute force documented in Chain 1) uses that access to create a new, fully 
privileged account — establishing a durable foothold independent of the originally 
compromised user. This maps to **MITRE ATT&CK T1136.002 (Create Account: Domain 
Account)** and **T1098 (Account Manipulation)**.

**Attacker:** Kali Linux (`10.10.5.12`), connected via RDP to DC01
**Target:** Domain Controller — DC01 (`10.10.5.10`)
**Detection:** Wazuh Manager (`10.10.5.20`)

---

### Step 1 — Remote access to the Domain Controller

Using `xfreerdp` from Kali, an RDP session was established directly against 
DC01:

\`\`\`bash
xfreerdp /v:10.10.5.10 /u:administrator /d:cyber.local /p:'<password>' /cert:ignore
\`\`\`

This simulates a scenario where an attacker has escalated from an initial 
foothold (e.g., the compromised standard user from Chain 1) to direct access on 
the domain controller itself — the highest-value target in the environment.

---

### Step 2 — Create a new domain account

From within the RDP session, PowerShell was used to create a new domain user 
account:

\`\`\`powershell
New-ADUser -Name "hacked" `
    -SamAccountName "hacked" `
    -UserPrincipalName "hacked@cyber.local" `
    -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $true
\`\`\`

---

### Step 3 — Escalate the new account to Domain Admins

The newly created account was then added to the **Domain Admins** group, granting 
it full administrative control over the domain:

\`\`\`powershell
Add-ADGroupMember -Identity "Domain Admins" -Members "hacked"
\`\`\`

Membership was confirmed with:

\`\`\`powershell
Get-ADGroupMember -Identity "Domain Admins"
\`\`\`

![scripts](images/scripts.png)

The hacked account can be view in AD Users & Computers on DC01:

![profile_view](images/profile_view.png)

---

### Step 4 — Detection in Wazuh

Both actions generated corresponding Windows Security Events, forwarded by the 
DC01 agent and surfaced in the Wazuh dashboard (**Threat Hunting → Events**):

![aadmin_logs](images/admin_logs.png)

Both events were captured using Wazuh's default ruleset with no custom rule 
authoring required, and included full subject/target account detail once the 
underlying event's full document view was expanded. If a SOC analyst were to 
see this in a real environment, it would immediately take top priority. In 
addition to being used as means for privilege escalation, this rogue account 
can also be used as a backdoor for an attacker.

---

### Observations
- This chain represents a meaningfully higher-severity scenario than Chain 1 
  alone: rather than just detecting unauthorized access, Wazuh captured the 
  attacker's follow-on action of establishing a second, fully privileged 
  identity — a common real-world persistence technique used to survive 
  password resets or account lockouts on the originally compromised credential.
- In a production environment, an alert on **any** addition to the Domain Admins 
  group is typically treated as a critical-severity event warranting immediate 
  investigation, given how few legitimate changes to that group should occur 
  outside of planned administrative activity.
- Combined, Chain 1 and Chain 2 demonstrate a full initial-access-to-domain- 
  compromise narrative, with detection coverage at each stage: brute-force 
  attempt, successful unauthorized logon, and privilege escalation.

---

---

## Chain 3: Kerberoasting

### Overview
This chain simulates a Kerberoasting attack, in which an already-compromised, 
low-privilege domain account is used to extract a crackable service ticket for 
a separate, higher-value service account — without requiring any additional 
privilege escalation to perform the extraction itself. This maps to 
**MITRE ATT&CK T1558.003 (Steal or Forge Kerberos Tickets: Kerberoasting)**.

**Attacker:** Kali Linux (`10.10.6.12`), authenticating as the previously 
compromised domain user Dante
**Target:** Domain Controller — DC01 (`10.10.5.10`)
**Detection:** Wazuh Manager (`10.10.5.20`)

---

### Step 1 — Creating a vulnerable service account

A domain service account was created on DC01 to represent a typical 
over-privileged, weakly-configured service account, a common real-world target 
for this attack. A Service Principal Name (SPN) was registered against the 
account, simulating a SQL Server instance — the step that makes the account 
eligible for a Kerberos service ticket request from any authenticated domain user:

![kerb_1](images/kerb_1.png)

---

### Step 2 — Requesting the ticket from Kali

Using Impacket's `GetUserSPNs.py`, the domain was enumerated for accounts with 
registered SPNs, and a service ticket was requested for `svc-sql`, 
authenticating as Dante — the same low-privilege account compromised in 
Chain 1:

![kerb_2](images/kerb_2.png)

The tool listed `svc-sql` and its SPN, and returned a Kerberos TGS-REP hash in 
the standard `$krb5tgs$` format, saved to `svc-sql.hash` for offline cracking.

**Notable finding:** no elevated privileges were required to perform this 
step. Kerberos service ticket requests are available by design to any 
authenticated domain user, regardless of their relationship to the target 
service account — this is the mechanism that makes Kerberoasting broadly 
exploitable from even a low-privilege starting foothold.

---

### Step 3 — Confirming the event on DC01

The resulting Kerberos service ticket request was visible in DC01's Event 
Viewer as **Event ID 4769** (A Kerberos service ticket was requested), logged 
under Windows Logs → Security.

![kerb_3](images/kerb_3.png)

---

### Step 4 — Locating the event in Wazuh

The event was located in the Wazuh dashboard (**Threat Hunting → Events**) by 
searching for the service account name directly:

\`\`\`
"svc-sql"
\`\`\`

Individual 4769 events were confirmed present and fully detailed in the raw 
event data, including:

- `data.win.system.eventID`: 4769
- `data.win.eventdata.serviceName`: svc-sql
- `data.win.eventdata.ticketEncryptionType`: encryption type used for the 
  ticket, varying between test runs (both AES and RC4 tickets were observed 
  across separate attempts)

![kerb_4](images/kerb_4.png)

**Detection observation:** unlike the RDP brute-force chain, a single 4769 
event for `svc-sql` is not, on its own, distinguishable from routine domain 
activity. Kerberos service ticket requests occur constantly as part of 
normal AD operation. No default Wazuh rule flags this event as suspicious, and 
a meaningful detection would need to correlate signals beyond a single log 
line: the volume and breadth of SPNs requested by one source in a short 
window, whether the requesting account has any legitimate relationship to the 
service, and the ticket's encryption type relative to what the domain 
otherwise supports.

---

### Step 5 — Cracking the extracted ticket

The extracted hash was cracked offline using Hashcat, selecting the mode 
matching the ticket's encryption type:

\`\`\`bash
hashcat -m 13100 -a 0 svc-sql.hash /usr/share/wordlists/rockyou.txt
\`\`\`

The cracked password was retrieved with:

![kerb_5](images/kerb_5.png)

The account's password (`P@ssw0rd123`) was successfully recovered, confirming 
the full attack path: a low-privilege compromised account was used to extract 
a service account's password hash without further exploitation, and that 
hash was subsequently cracked offline to obtain valid credentials for the 
service account.

---

### Observations
- This chain demonstrates a realistic privilege-widening path distinct from 
  Chain 2's direct privilege escalation — rather than granting Dante's 
  account more rights directly, the attacker pivots to compromising an 
  entirely separate, higher-value account via a design-level Kerberos 
  behavior rather than a vulnerability or misconfiguration in the traditional 
  sense.
- In a hardened production environment, this attack is more effectively 
  addressed through prevention than detection: strong, randomly-generated 
  service account passwords, Group Managed Service Accounts (gMSAs), and 
  disabling legacy RC4 support remove the exploitable weakness rather than 
  relying on catching the ticket request after the fact.
- This lab's service account was intentionally left in a default, 
  non-hardened configuration in order to study the attack's actual telemetry 
  and detection difficulty, rather than to model best-practice AD security.