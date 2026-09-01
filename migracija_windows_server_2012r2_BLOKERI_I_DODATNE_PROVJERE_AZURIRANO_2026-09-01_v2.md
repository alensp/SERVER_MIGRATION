# Blokeri i dodatne provjere prije migracije na novi DC
## Ažurirano 01.09.2026. — ADCS audit i konačna arhitektonska odluka

> **STATUS DOKUMENTA:** OVA DATOTEKA JE NOVA GLAVNA RADNA VERZIJA / SOURCE OF TRUTH ZA PROJEKT "vodoprivreda migracija".
>
> Nasljeđuje `migracija_windows_server_2012r2_BLOKERI_I_DODATNE_PROVJERE_AZURIRANO_2026-09-01.md`.
>
> Svi ranije potvrđeni AD/FSMO/DNS/DHCP/WINS/NetBIOS/SMB1 nalazi ostaju važeći.
> Ova verzija dodatno evidentira rezultat završnog ADCS source audita, uspješan CA backup,
> stvarnu HTTP CRL/CDP uporabu te konačnu odluku da zbog licenciranja NEĆE postojati treći
> Windows Server VM `CA01`. ADCS se dugoročno smješta na konačni Windows Server 2025 DC.

---

# 1. POTVRĐENO TRENUTNO STANJE

## Infrastruktura

```text
Domena:        vpi.local

VPI-SERVER
IP:            192.168.200.4
OS:            Windows Server 2012 R2
Status:        stari DC / aktivni ADCS CA / legacy workload host

VPI-HV01
IP:            192.168.200.5
OS:            Windows Server 2025 Standard
Status:        Hyper-V host

DC01
IP:            192.168.200.6
OS:            Windows Server 2022 Standard Evaluation
Status:        PRIVREMENI bridge DC

APP01
IP:            192.168.200.7
OS:            Windows Server 2025 Standard
Status:        APP / SQL / RDS / File cilj
```

## AD / FSMO

```text
[x] DC01 additional DC
[x] DC01 DNS + Global Catalog
[x] SYSVOL / NETLOGON zdravi
[x] AD replikacija VPI-SERVER <-> DC01 zdrava
[x] repadmin failures = 0
[x] svih 5 FSMO rola na DC01
[x] DC01 je PDC Emulator
[x] NTP novog PDC-a konfiguriran
```

## DHCP

```text
[x] DHCP migriran na DC01 24.08.2026.
[x] DC01 jedini autorizirani DHCP server
[x] stvarni DHCP lease testiran preko APP01
[x] VPI-SERVER deautoriziran kao DHCP server
```

Prijelazni DNS DHCP Option 006:

```text
192.168.200.6  DC01
192.168.200.4  VPI-SERVER
```

Ovisnost o `192.168.200.4` mora se ukloniti prije konačnog gašenja VPI-SERVER-a.

---

# 2. WINS / NETBIOS / SMB1 — ZAVRŠENA FAZA

## WINS

Potvrđeno:

```text
WINS Server feature         NIJE instaliran
WINS servis                 NE postoji
WINS registry konfiguracija NE postoji
WINS baza                   NE postoji
WINS replication partners   NE postoje
```

### ODLUKA

```text
[x] WINS se NE migrira.
[x] DC01 se NE pretvara u WINS server.
[x] Finalni DC2025 se NE pretvara u WINS server.
```

## NetBIOS

Identificirana broadcast NetBIOS imena:

```text
HP40D78F
CANON674DCF
HPB00CD140D78F
CANON661411
```

Pripadaju mrežnim printerima.

```text
[x] nema potvrđene kritične produkcijske ovisnosti koja blokira migraciju
[~] završni test pojedinačnih uređaja ostaje za decommission fazu
```

## SMB1

Na VPI-SERVER-u SMB1 postoji, ali audit nije dokazao aktivnu kritičnu udaljenu SMB1 sesiju.

```text
[x] SMB1 se NE migrira na novu infrastrukturu.
[x] SMB1 se NE uključuje na DC01.
[x] SMB1 se NE uključuje na finalni DC2025 bez eksplicitno dokazane potrebe.
```

## VPINASBackup

```text
VPINASBackup = QNAP NAS
namjena      = Veeam backup target
trenutno     = isključen
buduće       = planirano ponovno korištenje
```

Kod ponovnog uvođenja preferirati DNS/FQDN ili stabilnu IP konfiguraciju.

---

# 3. ADCS / CA — ZAKLJUČAK SOURCE AUDITA 01.09.2026.

Na VPI-SERVER-u izvršen je objedinjeni audit i backup:

```text
01_SOURCE_PRECHECK
02_SOURCE_USAGE_AND_WEB_AUDIT
02B_REVIEW_WEB_USAGE_RESULT
03_SOURCE_FULL_BACKUP
```

## 3.1. Aktivni CA

Potvrđeno:

```text
CA common name:      vpi-VPI-SERVER-CA-1
CA type:             Enterprise Root CA
CA server:           VPI-Server.vpi.local
CertSvc:             Running
UseDS:               1
CA enabled:          DA
Authenticated Users: Enroll dopušten
```

CA je aktivna produkcijska komponenta i NE smije se samo dekomisionirati.

```text
[x] CA SE MORA MIGRIRATI.
```

---

# 4. ADCS FULL BACKUP — ZAVRŠENO

Backup je 01.09.2026. uspješno napravljen.

Potvrđeno:

```text
CA database backup:
- certbkxp.dat
- edb log
- vpi-VPI-SERVER-CA-1.edb

Private key backup:
- vpi-VPI-SERVER-CA-1.p12

CertSvc nakon backupa:
Running
```

Lokalna validacija:

```text
Database backup present:       True
PFX/P12 key backup present:    True
Password file present:         True
LOCAL FILE VALIDATION:         PASS
```

### STOP/GO

```text
[x] SOURCE CA backup postoji.
[x] Private key backup postoji.
[x] CA database backup postoji.
[x] Source backup je dovoljno kompletan za nastavak pripreme migracije.

[!] Prije stvarnog cutovera i dalje je obavezan restore test / destination validation.
```

---

# 5. ADCS WEB / CRL / CDP USAGE AUDIT

IIS usage audit pregledao je 180 dana logova.

Rezultat:

```text
CertSrv Web Enrollment : 2 zahtjeva
CEP                    : 0 zahtjeva
CES                    : 0 zahtjeva
CertEnroll / CDP/CRL   : 1195 zahtjeva
```

## 5.1. CEP / CES

```text
[x] nema zabilježene CEP uporabe
[x] nema zabilježene CES uporabe
```

Na temelju dostupnih 180 dana IIS logova nema razloga planirati CEP/CES kao obaveznu
produkcijsku komponentu novog sustava.

## 5.2. /certsrv/

Zabilježena su samo:

```text
2 zahtjeva /certsrv/
```

To nije dokaz redovne produkcijske ovisnosti.

Web Enrollment se zasad NE smatra kritičnim workloadom, ali se ne uklanja sa source servera
dok ne završi cijela migracija.

## 5.3. /CertEnroll — KRITIČNA OVISNOST

Potvrđeno:

```text
1195 HTTP zahtjeva u 180 dana
```

Najčešće se dohvaćaju:

```text
/CertEnroll/vpi-VPI-SERVER-CA-1.crl
/CertEnroll/vpi-VPI-SERVER-CA-1+.crl
```

Klijenti uključuju više produkcijskih IP adresa iz `192.168.200.0/24`, kao i DC01.

User-Agent je u velikom broju slučajeva:

```text
Microsoft-CryptoAPI/10.0
```

### ZAKLJUČAK

```text
[!] http://VPI-Server/CertEnroll/ JE AKTIVNA PRODUKCIJSKA OVISNOST.
```

To nije samo konfiguracija koja postoji u certifikatima — klijenti stvarno redovno
dohvaćaju CRL/delta CRL preko tog URL-a.

---

# 6. CRL / AIA / LEGACY IME VPI-SERVER

Aktivni CA ima HTTP CDP:

```text
http://VPI-Server/CertEnroll/%3%8%9.crl
```

Postojeći izdani certifikati taj URL nose u sebi i on se retroaktivno ne može promijeniti.

Zato nakon uklanjanja starog VPI-SERVER hosta mora nastaviti raditi:

```text
http://VPI-Server/CertEnroll/
```

### ČVRSTO PRAVILO

```text
NE gasiti legacy ime VPI-Server bez zamjenskog HTTP CertEnroll endpointa.
NE uklanjati stare CRL / delta CRL / CA publication datoteke.
NE mijenjati CDP/AIA naslijepo prije restorea i testiranja CA-a.
```

Finalni dizajn mora omogućiti da DNS ime `VPI-Server` nakon decommissiona starog hosta
pokazuje na server koji može posluživati legacy `/CertEnroll` sadržaj.

---

# 7. AKTIVNA UPORABA CA-a

CA i dalje izdaje certifikate.

Potvrđeni noviji certifikati uključuju:

```text
Machine
Directory Email Replication
Domain Controller Authentication
Kerberos Authentication
```

DC01 je 21.08.2026. dobio:

```text
Machine
Directory Email Replication
Domain Controller Authentication
Kerberos Authentication
```

To potvrđuje da je Enterprise CA još aktivan u domain auto-enrollment procesu.

---

# 8. LICENCIRANJE — KONAČNA ODLUKA 01.09.2026.

Postojeća Windows Server 2025 Standard licenca koristi se za ciljnu arhitekturu s najviše
dva Windows Server VM-a na potpuno licenciranom Hyper-V hostu.

Planirani produkcijski VM-ovi su:

```text
VM 1 = finalni Windows Server 2025 DC
VM 2 = APP01 Windows Server 2025
```

Zaseban treći Windows Server VM:

```text
CA01
```

NEĆE se uvoditi jer za njega trenutno nema dodatne Windows Server licence.

### KONAČNA ODLUKA

```text
[!] DEDICATED CA01 OTPADA.

[x] ADCS konačno odredište =
    FINALNI DUGOROČNI WINDOWS SERVER 2025 DOMAIN CONTROLLER.
```

Ne koristiti:

```text
ADCS na APP01
ADCS na privremenom DC01 2022 Evaluation
```

---

# 9. KONAČNA CILJNA RASPODJELA ROLA

## VPI-HV01 — fizički host

```text
Windows Server 2025 Standard
Hyper-V host only
bez produkcijskih AD/DNS/DHCP/SQL/File/RDS/ADCS workloadova
```

## FINALNI DC2025 — VM 1

Konačno će sadržavati:

```text
AD DS
DNS
Global Catalog
FSMO svih 5 rola
PDC/NTP
DHCP
ADCS Certification Authority
potrebnu HTTP /CertEnroll funkciju za legacy CRL/CDP
```

## APP01 — VM 2

Konačno će sadržavati:

```text
File Server
shareove
DFS workload prema finalnom dizajnu
Folder Redirection podatke
SQL Server
poslovnu aplikaciju
RDS Session Host
aplikacijske IIS komponente
eventualni Print Server
backup/Veeam komponente prema finalnom dizajnu
```

## DC01 — samo privremeno

```text
Windows Server 2022 Evaluation
TEMPORARY BRIDGE DC
```

Dozvoljeno / trenutno postoji:

```text
AD DS
DNS
GC
FSMO
DHCP
```

Ne dodavati:

```text
ADCS
File
DFS workload
Print
SQL
RDS
business app
RD Gateway
aplikacijski IIS
Veeam
```

---

# 10. VAŽNA POSLJEDICA NOVE ADCS ODLUKE

Prethodni plan je imao ADCS cutover prije democije VPI-SERVER-a.

To sada NIJE izvedivo jer finalni ADCS host — finalni Windows Server 2025 DC — još ne postoji.

Istovremeno finalni Windows Server 2025 DC ne uvodimo dok postoji Windows Server 2012 R2 DC
i domena/forest su još na Windows2012R2 functional levelu.

Zato ADCS migracija mora biti izvedena kao pažljivo planirana prijelazna faza oko:

```text
VPI-SERVER 2012 R2 demotion
        ->
DFL/FFL 2016
        ->
uvođenje finalnog DC2025
        ->
ADCS restore/cutover na finalni DC2025
```

### KRITIČNA NAPOMENA

Stari VPI-SERVER se zbog aktivnog CA-a i `/CertEnroll` ovisnosti NE SMIJE jednostavno
ugasiti odmah nakon AD democije.

Mora ostati dostupan kao member server / CA host tijekom kratke prijelazne faze,
dok finalni DC2025 nije spreman preuzeti CA i legacy HTTP publication.

Drugim riječima:

```text
DEMOTION VPI-SERVER iz AD DS role != GAŠENJE VPI-SERVER-a.
```

To je sada jedna od najvažnijih migracijskih sigurnosnih točaka.

---

# 11. AŽURIRANI BLOKERI ZA POTPUNO GAŠENJE VPI-SERVER-a

## Završeno / nije blocker

```text
[x] dodatni DC uveden
[x] AD replikacija zdrava
[x] FSMO migriran na DC01
[x] DNS postoji na DC01
[x] DHCP migriran na DC01
[x] WINS migracija nije potrebna
[x] NetBIOS nije potvrđeni blocker
[x] SMB1 se ne migrira
[x] ADCS source audit napravljen
[x] ADCS CA potvrđen kao aktivan
[x] ADCS private-key backup napravljen
[x] ADCS CA database backup napravljen
[x] ADCS backup lokalno validiran PASS
[x] CEP usage audit = 0
[x] CES usage audit = 0
[x] CertSrv usage = minimalan
[x] /CertEnroll usage = potvrđeno aktivan
[x] konačni ADCS host odlučen = finalni DC2025
```

## Otvoreno

```text
[ ] File Server / svi shareovi
[ ] DFS Namespace
[ ] sve UNC reference \\VPI-SERVER\
[ ] Folder Redirection / GPO reference
[ ] SQL
[ ] poslovna aplikacija
[ ] RDS
[ ] RD Gateway / aplikacijski IIS
[ ] printeri / Print Server
[ ] backup / Veeam finalni dizajn
[ ] DHCP Option 006 ukloniti ovisnost o 192.168.200.4
[ ] završni NetBIOS/SMB1 uređaj test

[ ] finalni pre-demotion AD health audit
[ ] demotirati VPI-SERVER iz DC role
[ ] podići DFL/FFL na Windows Server 2016
[ ] napraviti finalni Windows Server 2025 DC VM
[ ] promovirati finalni DC2025
[ ] migrirati FSMO/DNS/DHCP/NTP na finalni DC2025
[ ] migrirati/restoreati ADCS na finalni DC2025
[ ] osigurati legacy http://VPI-Server/CertEnroll/
[ ] provesti CA restore/cutover validation
[ ] ukloniti DC01 2022 Evaluation
[ ] tek tada potpuno ugasiti/decommissionirati VPI-SERVER
```

---

# 12. NOVI DOGOVORENI OPERATIVNI REDOSLIJED

## FAZA 1 — ADCS SOURCE AUDIT I BACKUP

```text
[x] 01_SOURCE_PRECHECK
[x] 02_SOURCE_USAGE_AND_WEB_AUDIT
[x] 02B_REVIEW_WEB_USAGE_RESULT
[x] 03_SOURCE_FULL_BACKUP

[x] CA aktivan
[x] backup PASS
[x] /CertEnroll aktivan
[x] CEP = 0
[x] CES = 0
[x] destination odlučen = finalni DC2025
```

### STATUS

```text
FAZA 1 ZAVRŠENA.
```

---

## FAZA 2 — SLJEDEĆE: FILE / DFS / UNC / GPO AUDIT

Ovo je sada neposredno sljedeći radni korak.

Napraviti read-only inventuru VPI-SERVER-a:

```text
1. svi SMB shareovi
2. share permissions
3. NTFS permissions
4. veličine i lokacije podataka
5. DFS Namespace konfiguracija
6. DFS folder targets
7. Folder Redirection GPO
8. sve GPO reference na \\VPI-SERVER\
9. logon/startup skripte
10. mapped drives
11. aplikacijske UNC reference
12. Scheduled Tasks / servisi koji koriste UNC putanje
13. printer shareovi ako postoje
14. procjena što ide direktno na APP01
15. plan očuvanja starih UNC imena ako ih aplikacije zahtijevaju
```

Cilj:

```text
potpuno odvojiti File/DFS workload od VPI-SERVER-a
i migrirati ga direktno na APP01.
```

---

## FAZA 3 — SQL / POSLOVNA APLIKACIJA / RDS AUDIT I MIGRACIJA NA APP01

Inventura:

```text
SQL instance
databases
logins
SQL Agent jobs
linked servers
ODBC
service accounts
connection strings
application services
scheduled tasks
RDS roles
RDS licensing
collections
RD Gateway dependencies
certificates
```

Cilj:

```text
SQL + business app + RDS direktno VPI-SERVER -> APP01.
```

---

## FAZA 4 — PRINT / IIS / RD GATEWAY / BACKUP

```text
printer queues
printer drivers
direct-IP/DNS mogućnosti
IIS sites
bindings
certifikati
RD Gateway
Remote Web komponente
Veeam
VPINASBackup/QNAP
```

---

## FAZA 5 — FINALNI PRE-DEMOTION AUDIT VPI-SERVER-a

Prije democije iz AD DS role mora vrijediti:

```text
[ ] File/DFS workload riješen
[ ] SQL/App/RDS workload riješen
[ ] Print/IIS/Gateway workload riješen
[ ] backup/Veeam riješen
[ ] DHCP/DNS klijenti ne ovise kritično o .4 kao DNS serveru
[ ] AD replication healthy
[ ] dcdiag PASS
[ ] repadmin failures = 0
[ ] FSMO ostaje na DC01
[ ] CA backup ponovno potvrđen
[ ] VPI-SERVER može ostati member server s aktivnim CA-om tijekom prijelaza
```

---

## FAZA 6 — DEMOTION VPI-SERVER 2012 R2 IZ DOMAIN CONTROLLER ROLE

```text
1. finalni dcdiag
2. finalni repadmin
3. demotirati VPI-SERVER iz DC role
4. NE gasiti server
5. ostaviti ga privremeno kao domain member / CA host
6. potvrditi DNS cleanup
7. potvrditi AD Sites cleanup
8. potvrditi DC01 kao jedini zdravi DC/GC
9. potvrditi da CA i http://VPI-Server/CertEnroll i dalje rade
```

### STOP

```text
Ako ADCS/CertEnroll ne radi nakon democije:
NE nastavljati na DFL/FFL dok problem nije razumljiv i riješen.
```

---

## FAZA 7 — RAISE DOMAIN / FOREST FUNCTIONAL LEVEL

Tek kad Windows Server 2012 R2 više nije Domain Controller:

```text
Domain Functional Level -> Windows Server 2016
Forest Functional Level -> Windows Server 2016
```

Prije i poslije:

```text
dcdiag
repadmin /replsummary
repadmin /showrepl
DNS provjere
SYSVOL/NETLOGON
```

---

## FAZA 8 — KREIRATI FINALNI WINDOWS SERVER 2025 DC VM

Ovo je VM koji koristi prvo trajno Windows Server virtualizacijsko pravo.

Koraci:

```text
1. kreirati novi Windows Server 2025 VM
2. odabrati finalni hostname
3. dodijeliti finalni statički IP
4. DNS privremeno usmjeriti na DC01
5. join vpi.local
6. instalirati AD DS + DNS
7. promovirati kao additional DC
8. uključiti Global Catalog
9. provjeriti SYSVOL / NETLOGON
10. dcdiag
11. repadmin /replsummary
12. repadmin /showrepl
13. DNS testovi
```

Ne instalirati ADCS dok novi DC ne prođe sve health provjere.

---

## FAZA 9 — FINALIZIRATI INFRASTRUKTURNE ROLE NA DC2025

Nakon zdravog AD/DC stanja:

```text
1. prebaciti svih 5 FSMO rola sa DC01
2. konfigurirati PDC/NTP
3. migrirati DHCP sa DC01
4. finalizirati DHCP Option 006
5. ukloniti DNS dependency na 192.168.200.4
6. potvrditi klijentske DHCP/DNS testove
```

---

## FAZA 10 — ADCS MIGRACIJA NA FINALNI DC2025

Tek sada:

```text
1. napraviti novi finalni source CA backup neposredno prije cutovera
2. provjeriti .p12 / CA database / registry
3. zaustaviti issuance na source CA u kontroliranom cutoveru
4. instalirati ADCS Certification Authority na finalni DC2025
5. restoreati postojeći CA identitet
6. restoreati CA database
7. restoreati registry/config prema verificiranom migration planu
8. pokrenuti CertSvc
9. potvrditi CA identity / chain / templates / issuance
10. generirati/potvrditi CRL i delta CRL
11. osigurati legacy /CertEnroll publication
12. osigurati da `http://VPI-Server/CertEnroll/` radi
13. testirati certutil URL retrieval
14. testirati novi auto-enrollment
15. provjeriti DC certifikate
16. provjeriti IIS logove / CRL dohvat
```

### ČVRSTI STOP

```text
NE uklanjati source CA dok destination CA nije potpuno validiran.
```

---

## FAZA 11 — UKLONITI DC01 2022 EVALUATION

Tek kada finalni DC2025 ima:

```text
[x/ ] AD DS
[x/ ] DNS
[x/ ] GC
[x/ ] FSMO
[x/ ] DHCP
[x/ ] PDC/NTP
[x/ ] ADCS
```

i svi health testovi prolaze:

```text
1. demote DC01
2. DNS/AD cleanup
3. ugasiti i ukloniti Evaluation VM
```

---

## FAZA 12 — KONAČNI DECOMMISSION VPI-SERVER

Tek kad:

```text
[ ] ADCS je potpuno na finalnom DC2025
[ ] legacy /CertEnroll radi bez starog hosta
[ ] File/DFS je na APP01
[ ] SQL/App/RDS je na APP01
[ ] DNS/DHCP više ne ovise o .4
[ ] backup radi
[ ] printeri rade
[ ] završni NetBIOS/SMB1 test završen
```

onda:

```text
1. ukloniti preostale source CA role ako ih još ima
2. osloboditi staro hostname/IP prema DNS/CDP planu
3. postaviti potreban DNS alias za VPI-Server
4. provjeriti /CertEnroll preko legacy imena
5. ugasiti stari Windows Server 2012 R2
6. zadržati rollback backup prema dogovorenom retentionu
```

---

# 13. AKTIVNA FAZA RADA — STAVKA 1: DOVRŠITI MIGRACIJU WORKLOADOVA SA VPI-SERVER-a

> **STATUS 01.09.2026.: KREĆEMO DALJE SA STAVKOM 1.**
>
> Cilj ove faze je ukloniti sve produkcijske workloadove sa `VPI-SERVER` koji nisu ADCS,
> tako da se kasnije server može sigurno demotirati iz Domain Controller role.

## 13.1. File / DFS / UNC / GPO -> APP01

**OVO JE NEPOSREDNO SLJEDEĆI OPERATIVNI KORAK.**

Napraviti prvo read-only audit, zatim plan i migraciju:

```text
[ ] svi SMB shareovi
[ ] share permissions
[ ] NTFS permissions
[ ] veličine i fizičke lokacije podataka
[ ] DFS Namespace konfiguracija
[ ] DFS folder targets
[ ] Folder Redirection GPO
[ ] sve GPO reference na \\VPI-SERVER\
[ ] logon / startup / shutdown skripte
[ ] mapped drives
[ ] aplikacijske UNC reference
[ ] Scheduled Tasks koji koriste \\VPI-SERVER\...
[ ] servisi koji koriste \\VPI-SERVER\...
[ ] printer shareovi ako postoje
[ ] plan migracije podataka na APP01
[ ] plan očuvanja starih UNC imena ako ih aplikacije zahtijevaju
```

Cilj:

```text
File / DFS / UNC / GPO workload -> APP01
```

Ne mijenjati produkciju dok audit nije pregledan.

---

## 13.2. SQL / poslovna aplikacija / RDS -> APP01

Nakon File/DFS faze:

```text
[ ] SQL instance inventory
[ ] baze
[ ] logins
[ ] SQL Agent jobs
[ ] linked servers
[ ] ODBC
[ ] service accounts
[ ] connection strings
[ ] aplikacijski servisi
[ ] Scheduled Tasks
[ ] RDS role
[ ] RDS licensing
[ ] collections
[ ] certifikati
[ ] aplikacijski IIS ako je vezan uz poslovnu aplikaciju
```

Cilj:

```text
SQL + poslovna aplikacija + RDS -> APP01
```

---

## 13.3. Print / IIS / backup prema konačnom planu

Nakon glavnih workloadova:

```text
[ ] printer queues / drivers
[ ] procjena direct-IP/DNS printer konfiguracije
[ ] IIS sites / bindings / certifikati
[ ] RD Gateway / Remote Web ako postoji produkcijska ovisnost
[ ] Veeam / backup komponente
[ ] VPINASBackup ponovno uvođenje prema finalnom backup planu
```

---

## 13.4. ADCS zasad ostaje na VPI-SERVER-u

**ADCS se u ovoj fazi NE SELI.**

```text
[x] ADCS source audit završen
[x] CA backup PASS
[x] /CertEnroll aktivna produkcijska ovisnost potvrđena

[!] ADCS ostaje na VPI-SERVER-u
    sve dok:
    - VPI-SERVER nije demotiran iz DC role,
    - DFL/FFL nisu podignuti na Windows Server 2016,
    - finalni Windows Server 2025 DC nije uveden i potvrđeno zdrav,
    - destination ADCS restore/cutover nije spreman.
```

---

## 13.5. GO kriterij za izlazak iz STAVKE 1

Stavka 1 je završena tek kada vrijedi:

```text
[ ] File/DFS/UNC/GPO workload riješen i prebačen na APP01
[ ] SQL/aplikacija/RDS riješeni i prebačeni na APP01
[ ] Print/IIS/backup riješeni prema finalnom planu
[x] ADCS namjerno ostaje na VPI-SERVER-u kao jedini preostali privremeni workload
```

Tek tada prelazimo na:

```text
FINALNI PRE-DEMOTION AD HEALTH AUDIT
        ->
DEMOTION VPI-SERVER-a IZ DOMAIN CONTROLLER ROLE
        ->
DFL/FFL 2016
        ->
FINALNI WINDOWS SERVER 2025 DC
        ->
FSMO/DNS/DHCP/NTP
        ->
ADCS MIGRACIJA
        ->
UKLANJANJE DC01 2022 EVALUATION
```

---

# 14. TRENUTNO SLJEDEĆI KORAK — 01.09.2026.

```text
STAVKA 1 JE AKTIVNA.

SADA RADIMO:
FILE / DFS / UNC / GPO READ-ONLY AUDIT NA VPI-SERVER-u.

NAKON TOGA:
SQL / aplikacija / RDS -> APP01
Print / IIS / backup -> finalni plan

ADCS ZA SADA OSTAJE NA VPI-SERVER-u.
```

Za sada:

```text
NE raditi ADCS cutover.
NE demotirati VPI-SERVER.
NE podizati DFL/FFL.
NE kreirati/promovirati finalni DC2025 još.
NE dodavati ADCS na DC01.
NE dodavati ADCS na APP01.
```

# 14. KONAČNI SAŽETAK STRATEGIJE

```text
VPI-SERVER 2012 R2
    |
    +-- AD/DNS/FSMO/DHCP ------> DC01 2022 Evaluation   [bridge, već gotovo]
    |
    +-- File/DFS -------------->
    |                           APP01 2025
    +-- SQL/App/RDS -----------> APP01 2025
    |
    +-- Print/IIS/Gateway -----> APP01 / finalni dizajn
    |
    +-- ADCS ostaje PRIVREMENO na VPI-SERVER-u
    |   dok finalni DC2025 ne postoji
    |
    +-- DEMOTION IZ DC ROLE
            |
            |  VPI-SERVER ostaje privremeno živ kao CA/member server
            v
    DFL/FFL -> Windows Server 2016
            |
            v
    FINALNI WINDOWS SERVER 2025 DC
            |
            +-- AD DS / DNS / GC
            +-- FSMO / DHCP / NTP
            +-- ADCS
            +-- legacy /CertEnroll
            |
            v
    ukloniti DC01 2022 Evaluation
            |
            v
    konačno ugasiti VPI-SERVER
```

**Ovaj redoslijed je od 01.09.2026. novi glavni operativni plan projekta.**
