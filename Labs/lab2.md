## Laboratorul 2: Securizarea routerului pentru acces administrativ

---

## Cuprins

1. [Cerințe generale și topologie](#cerințe-generale-și-topologie)  
2. [Partea 1: Configurarea de bază a dispozitivului de rețea](#partea-1-configurarea-de-bază-a-dispozitivului-de-rețea)  
   1. [Pasul 1: Conectarea fizică și accesul la consola routerului](#pasul-1-conectarea-fizică-și-accesul-la-consola-routerului)  
   2. [Pasul 2: Configurarea de bază (hostname, parole)](#pasul-2-configurarea-de-bază-hostname-parole)  
   3. [Pasul 3: Configurarea adreselor IP pe routere și PC-uri](#pasul-3-configurarea-adreselor-ip-pe-routere-și-pc-uri)  
   4. [Pasul 4: Configurarea rutării statice](#pasul-4-configurarea-rutării-statice)  
   5. [Pasul 5: Verificarea conectivității](#pasul-5-verificarea-conectivității)  
3. [Partea 2: Control acces administrativ pentru routere](#partea-2-control-acces-administrativ-pentru-routere)  
   1. [Pasul 1: Configurarea și criptarea parolelor](#pasul-1-configurarea-și-criptarea-parolelor)  
   2. [Pasul 2: Configurarea bannerelor](#pasul-2-configurarea-bannerelor)  
   3. [Pasul 3: Configurarea liniilor VTY și a autentificării](#pasul-3-configurarea-liniilor-vty-și-a-autentificării)  
   4. [Pasul 4: Configurarea SSH](#pasul-4-configurarea-ssh)  
4. [Partea 3: Configurarea rolurilor administrative (privilegii multiple)](#partea-3-configurarea-rolurilor-administrative-privilegii-multiple)  
5. [Partea 4: Configurarea raportărilor de reziliență și management (NTP, Syslog, SNMP)](#partea-4-configurarea-raportărilor-de-reziliență-și-management-ntp-syslog-snmp)  
6. [Partea 5: Configurarea securității automate (AutoSecure și auditul de securitate)](#partea-5-configurarea-securității-automate-autosecure-și-auditul-de-securitate)  
7. [Verificări finale](#verificări-finale)  

---

## Cerințe generale și topologie

- **Echipamente recomandate**:  
  - 3 x Routere Cisco 1841 (sau similare) cu IOS 12.4(20) T1 (sau compatibil) și posibil cu SDM 2.5 instalat.  
  - 2 x Switch-uri Cisco 2960 (sau comparabile).  
  - PC-uri (Windows XP/Vista/7/Server) cu PuTTY SSH Client sau alt software terminal.  
  - Cabluri serial și Ethernet (după topologia din laborator).  
  - Cabluri rollover pentru conectarea prin portul de consolă la router.

- **Topologie minimă** (exemplu):
  ```
  [PC-A] --(Fa0/0) R1 (Fa0/1) -- Switch --(Fa0/0) R2 (Fa0/1) -- Switch -- (Fa0/0) R3 -- [PC-C]
               |                                         |
               (Serial0/0/0) <----------- WAN ----------> (Serial0/0/0)
  ```
  Adaptați porturile și interfețele în funcție de echipamentele voastre.

---

## Partea 1: Configurarea de bază a dispozitivului de rețea

### Pasul 1: Conectarea fizică și accesul la consola routerului

1. **Cablare**: Conectați routerele la switch-uri conform schemei. Interfețele serial între routere vor fi conectate cu un cablu DCE/DTE (un capăt setat ca DCE, celălalt ca DTE).  
2. **Port consolă**: Utilizați cablul rollover pentru a vă conecta de la portul serial al PC-ului (prin adaptor, dacă e cazul) la portul de consolă al routerului.  
3. **Acces**: Deschideți un client terminal (ex: PuTTY, TeraTerm) și setați:  
   - `Speed: 9600`  
   - `Data bits: 8`  
   - `Parity: None`  
   - `Stop bits: 1`  
   - `Flow control: None`  

### Pasul 2: Configurarea de bază (hostname, parole)

De exemplu, pentru **R1**:

```bash
Router> enable
Router# configure terminal
Router(config)# hostname R1

! Configurare parola pentru modul EXEC privilegiat
R1(config)# enable secret cisco123

! Parola consolei
R1(config)# line console 0
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! Parola pentru linia aux (dacă există acces)
R1(config)# line aux 0
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! Salvare configurație
R1(config)# end
R1# copy running-config startup-config
```

> Repetați pentru fiecare router (R2, R3) cu hostnames și parole proprii.

### Pasul 3: Configurarea adreselor IP pe routere și PC-uri

#### Exemplar R1

Să presupunem că interfața `Fa0/0` a lui R1 merge spre PC-A și are adresa `192.168.1.1/24`, iar PC-A are `192.168.1.10/24`.

```bash
R1# configure terminal
R1(config)# interface fastEthernet 0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

Dacă R1 are legătură serială spre R2:

```bash
R1(config)# interface serial 0/0/0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# clock rate 64000      ! Puneți asta pe DCE
R1(config-if)# no shutdown
R1(config-if)# exit
```

> **Notă**: Pe un router, interfata serială ce are cablul DCE cere `clock rate`. Pe celălalt (DTE), nu.

#### Configurare PC

Pe PC-A:
1. Accesați **Network Connections** → **Properties** → **TCP/IP v4**.
2. Setați IP `192.168.1.10`, Mask `255.255.255.0`, Gateway `192.168.1.1`.

Repetați procesul pentru R2, R3 și PC-urile aferente subrețelelor.

### Pasul 4: Configurarea rutării statice

#### Exemplar

- R1 știe direct despre `192.168.1.0/24` și `10.10.10.0/30`. Pentru a ajunge la R3, să presupunem că e o altă rețea, `192.168.3.0/24`, conectată la R3. R1 trebuie să știe cum să ajungă acolo:

```bash
R1(config)# ip route 192.168.3.0 255.255.255.0 10.10.10.2
```

- R2, care este în mijloc, știe direct despre `10.10.10.0/30` și, să zicem, `10.10.20.0/30`. Va avea rute pentru `192.168.1.0/24` și `192.168.3.0/24`. Exemplu:

```bash
R2(config)# ip route 192.168.1.0 255.255.255.0 10.10.10.1
R2(config)# ip route 192.168.3.0 255.255.255.0 10.10.20.2
```

- R3 cunoaște direct `192.168.3.0/24` și `10.10.20.0/30`. Are nevoie de rută spre `192.168.1.0/24`:

```bash
R3(config)# ip route 192.168.1.0 255.255.255.0 10.10.20.1
```

> **Notă**: Ajustați IP-urile și masca după cum v-ați definit topologia.  

### Pasul 5: Verificarea conectivității

1. **Ping** între PC-A și gateway-ul R1 (ex: `ping 192.168.1.1`).
2. **Ping** R1 → R2 → R3 pe interfețele lor IP de Serial/Ethernet.
3. **Ping** PC-A → PC-C (trecând prin cele 3 routere).

Dacă totul este configurat corect, veți primi răspuns la ping.

---

## Partea 2: Control acces administrativ pentru routere

### Pasul 1: Configurarea și criptarea parolelor

1. **Activarea criptării parolelor la vedere**:

```bash
R1(config)# service password-encryption
```

2. **Verificați** că parolele din config (show running-config) nu mai apar în clar.

### Pasul 2: Configurarea bannerelor

```bash
R1(config)# banner motd #
ATENTIE: Acces restrictionat! 
Orice acces neautorizat este interzis.
#
```

> Se recomandă folosirea unui mesaj care avertizează clar despre natura securizată a sistemului.

### Pasul 3: Configurarea liniilor VTY și a autentificării

```bash
R1(config)# line vty 0 4
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# transport input telnet ssh
R1(config-line)# exit
```

> În practică, se recomandă **dezactivarea** Telnet, permițând doar SSH. Vom face asta la pasul următor.

### Pasul 4: Configurarea SSH

1. **Setați un nume de domeniu** (obligatoriu pentru a genera cheile RSA):

```bash
R1(config)# ip domain-name cisco.local
```

2. **Generați cheile RSA**:

```bash
R1(config)# crypto key generate rsa
% Key size: 1024 (or 2048)
```

3. **Creați un user pentru autentificare locală**:

```bash
R1(config)# username admin privilege 15 secret admin123
```

4. **Configurați linia VTY pentru SSH și autentificare locală**:

```bash
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# login local
R1(config-line)# exit
```

5. **Verificați**:  
   - Din PC, încercați `ssh admin@192.168.1.1` (sau IP-ul routerului).  
   - Ar trebui să vi se ceară parola `admin123`.

---

## Partea 3: Configurarea rolurilor administrative (privilegii multiple)

Pentru a defini **niveluri de privilegii diferite** (exemplu, un utilizator “read-only” vs. un utilizator “admin”):

1. **Creare user cu privilegiu 1**:

```bash
R1(config)# username readOnly privilege 1 secret readonly123
```

2. **Creare user cu privilegiu 15**:

```bash
R1(config)# username superAdmin privilege 15 secret super123
```

3. **Comenzi custom pentru privilegii** (opțional, dacă doriți să restricționați comenzi specifice).  

4. **Testare**:  
   - Vă logați cu user `readOnly` și observați că nu puteți intra în config.  
   - Vă logați cu `superAdmin` și puteți face tot.

---

## Partea 4: Configurarea raportărilor de reziliență și management (NTP, Syslog, SNMP)

### Configurarea unui router ca sursă de timp (NTP)

1. **Pe routerul R1** (dacă îl considerați server NTP):

```bash
R1(config)# ntp master 3
```

2. **Pe R2 și R3**:

```bash
R2(config)# ntp server 10.10.10.1
R3(config)# ntp server 10.10.20.1
```

> IP-urile folosite ca server NTP pot fi interfețele routerului R1, adaptat la topologia voastră.

### Configurarea Syslog

1. **Instalați un server Syslog** (de ex. Kiwi Syslog sau Tftpd32 pe PC). Notați IP-ul PC-ului (ex. `192.168.1.100`).  
2. **Pe router**:

```bash
R1(config)# logging host 192.168.1.100
R1(config)# logging trap informational
```

3. **Test**: Generați evenimente (ex. `shutdown/no shutdown` pe o interfață) și vedeți dacă apar logurile pe serverul Syslog.

### Configurarea SNMP

1. **Comunitate SNMP read-only**:

```bash
R1(config)# snmp-server community public RO
```

2. **Comunitate SNMP read-write** (atenție la securitate):

```bash
R1(config)# snmp-server community private RW
```

3. **Setări de contact, location** (opțional):

```bash
R1(config)# snmp-server contact AdminLab
R1(config)# snmp-server location Sala-IT
```

4. **Pe PC**, puteți folosi un tool gen `SNMPc` sau alt manager SNMP pentru a monitoriza routerul.

### Configurarea rapoartelor de capcane (traps)

```bash
R1(config)# snmp-server enable traps
R1(config)# snmp-server host 192.168.1.100 version 2c public
```

---

## Partea 5: Configurarea securității automate (AutoSecure și auditul de securitate)

### AutoSecure

1. **Rulați comanda**:

```bash
R1# auto secure
```

2. **Urmați pașii** indicați de wizard (vi se va cere să dezactivați servcii nefolosite, să configurați firewall CBAC, etc.).  

3. **La final**, comanda va afișa ce setări au fost modificate.

### Security Audit (SDM)

1. **Lansați Cisco SDM** (dacă routerul are SDM instalat).  
2. **Navigați** la secțiunea `Configure` → `Security Audit` sau `One-Step Lockdown`.  
3. **Analizați** vulnerabilitățile și aplicați recomandările.  
4. **Confirmați** că accesul la router rămâne funcțional (SSH/Telnet, după cum e cazul).

---

## Verificări finale

1. **Verificați** că puteți face SSH pe router.  
2. **Verificați** bannerele la conectare.  
3. **Verificați** că parolele nu apar în clar (`show run`).  
4. **Verificați** logurile Syslog pe server (dacă ați făcut evenimente).  
5. **Verificați** orarul curent (`show clock`) pe fiecare router – să fie sincronizat NTP.  
6. **SNMP**: folosiți un manager extern și asigurați-vă că routerul răspunde la GET.

---

## Încheiere

- **Salvați** configurația pe fiecare router:

  ```bash
  R1# copy running-config startup-config
  ```

- **Documentați** eventualele observații și particularități.  
- **Fișierele de configurare finală** pot fi atașate raportului/laboratorului pentru evaluare.

---

> Acest laborator acoperă configurări cheie pentru securizarea accesului administrativ pe router, implicând parole, criptare, SSH, bannere de avertizare, protecție cu SNMP, Syslog și NTP, precum și modul de automatizare a securității prin AutoSecure și instrumente SDM. Adaptați valorile (IP-uri, parole, nume de domenii) la mediul vostru.  
---
