## LAB 5: Configurarea unui sistem de prevenire a intruziunilor (IPS) folosind CLI și SDM

## Obiective

1. **Partea 1: Configurare de bază a routerului**  
   - Configurați numele de gazdă, adresele IP ale interfețelor și parolele de acces.  
   - Configurați rutarea statică (sau dinamică) astfel încât să existe conectivitate între dispozitivele din rețea.  

2. **Partea 2: Configurarea unui sistem de prevenire a intruziunilor IOS (IPS) utilizând CLI**  
   - Configurați **IPS IOS** din CLI (activare, definire directoare, import semnături).  
   - Modificați semnături IPS (activare/dezactivare, modificare acțiuni).  
   - Examinați configurația finală a IPS (comenzi *show*).  
   - Verificați funcționalitatea IPS (simulând un atac cu un instrument de scanare).  
   - Redirecționați mesajele IPS către un server Syslog.  

3. **Partea 3: Configurarea unui sistem de prevenire a intruziunilor (IPS) folosind SDM**  
   - Configurați **IPS** prin Cisco SDM (Wizard IPS).  
   - Modificați semnăturile IPS (politici de reacție, severitate).  
   - Examinați configurația rezultat.  
   - Folosiți un instrument de scanare (ex. *SuperScan*, *Nmap*) pentru a simula un atac.  
   - Utilizați modulul de monitorizare SDM pentru a observa alertele și a verifica funcționalitatea IPS.  

---

## Resurse necesare

- **2 routere** (R1 și R3) cu **SDM 2.5** instalat (Cisco 1841 cu IOS 12.4(20) T1 și >=192 MB DRAM).  
- **1 router** (R2) cu versiune compatibilă de IOS (12.4(20) T1) – opțional, pentru rutare intermediară.  
- **2 comutatoare** (Cisco 2960 sau echivalent).  
- **PC-A** și **PC-C** (Windows XP/Vista/7):  
  - *Syslog server* (ex. Kiwi, Tftpd32)  
  - *TFTP server* (pentru încărcarea semnăturilor IPS)  
  - *Instrument de scanare* (ex. SuperScan, Nmap)  
  - *Java* (minim v6) pentru rularea SDM (și alocare memorie de 256 MB, dacă e cazul)  
- **Cabluri serial și Ethernet** conform topologiei.  
- **Cabluri rollover** pentru a configura routerele prin portul de consolă.  
- *Pachetul de semnături IPS* (de ex. `signature.tar`) și fișierele publice cu cheie crypto, furnizate de instructor/laborator.

> **Notă**: Pentru a putea activa IPS pe R1 și R3, trebuie să aveți suficientă memorie DRAM și Flash.  
> **Notă**: Asigurați-vă că routerele/comutatoarele nu au configurații vechi salvate (folosiți `erase startup-config` și `reload`).

---

## Partea 1: Configurare de bază a routerului

### 1.1 Acces prin consolă și setări inițiale

1. **Conectați** PC-ul la portul *Console* al routerului (cablul rollover + adaptor)  
2. **Deschideți** un terminal (PuTTY): 9600 baud, 8N1, no flow control.  
3. **Introduceți** comenzi de configurare:

Exemplu, pe **R1**:

```bash
Router> enable
Router# configure terminal
Router(config)# hostname R1

! Parola enable
R1(config)# enable secret cisco123

! Linie de console
R1(config)# line console 0
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! Linie VTY
R1(config)# line vty 0 4
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! Activare criptare parole
R1(config)# service password-encryption

! Banner
R1(config)# banner motd # Acces restrictionat! #

! Interfețe
R1(config)# interface fastEthernet 0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

! (opțional) Interfață Serial dacă aveți conexiuni WAN
R1(config)# interface serial 0/0/0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# clock rate 64000
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# end
R1# copy running-config startup-config
```

**Repetați** pașii similari pentru R3. R2, dacă este intermediar, poate fi configurat cu rutare.

### 1.2 Rutare statică (sau dinamică)

- Dacă topologia e mică, puteți folosi *ip route* statice.  
- Dacă e complexă, folosiți un protocol dinamic (ex. EIGRP/OSPF).  

Exemplu static:

```bash
R1(config)# ip route 192.168.2.0 255.255.255.0 10.10.10.2
```

---

## Partea 2: Configurarea unui sistem de prevenire a intruziunilor IOS (IPS) utilizând CLI

### 2.1 Concept

Cisco IOS IPS monitorizează traficul și, pe baza unor semnături, poate **alerte** sau **bloca** atacuri cunoscute. Pentru a funcționa:
1. Se **activează** funcționalitatea IPS pe router (depinde de licență/versiune).  
2. Se **creează** un director în flash pentru semnături.  
3. Se **copiază** fișierul cu semnături (`signature.tar`) de pe TFTP server în flash.  
4. Se **configurează** un *IPS rule* (ex. `ip ips name <rule-name>`).  
5. Se **atașează** IPS rule la interfață (ex. `ip ips <rule-name> in`).

### 2.2 Pași de configurare

#### Pasul 1: Crearea unui director IPS pe flash

```bash
R1# mkdir flash:ips
```
Verificați cu:
```bash
R1# dir flash:
```

#### Pasul 2: Copierea semnăturilor IPS

Presupunem că pe PC-A rulați un server TFTP la IP `192.168.1.100`, iar fișierul semnăturilor este `IPS-Signature.tar`.

```bash
R1# copy tftp: flash:/ips
Address or name of remote host? 192.168.1.100
Source filename? IPS-Signature.tar
Destination filename? IPS-Signature.tar
```
Așteptați finalizarea copiei. Apoi verificați:

```bash
R1# dir flash:/ips
```

#### Pasul 3: Importarea semnăturilor în config IOS

```bash
R1# configure terminal
! Indicați locația semnăturilor și a fișierelor de chei publice
R1(config)# ip ips sdf location flash:/ips
R1(config)# ip ips name MYIPS
R1(config-ips)# signature-definition
R1(config-ips-sign-def)# signature memory-size 512
R1(config-ips-sign-def)# load flash:/ips/IPS-Signature.tar
R1(config-ips-sign-def)# exit
R1(config-ips)# exit
```

> `signature memory-size 512` este un exemplu; ajustați în funcție de memorie.

#### Pasul 4: Activarea IPS pe interfață

Să zicem că interfața *Fa0/0* e considerată “internă”, iar traficul care intră este supus inspecției:

```bash
R1(config)# interface fastEthernet 0/0
R1(config-if)# ip ips MYIPS in
R1(config-if)# exit
```
> `in` = inspecție a traficului care sosește pe Fa0/0. De asemenea, puteți folosi `out` pe altă interfață, în funcție de topologia voastră.

#### Pasul 5: (Opțional) Salvarea și ajustarea semnăturilor

Dacă doriți să dezactivați anumite semnături sau să schimbați acțiunile (alarm, drop, reset), o puteți face tot în subconfigurația `ip ips name MYIPS`. Comanda tipică este:

```bash
R1(config)# ip ips name MYIPS
R1(config-ips)# event-action produce-alert
! Alte acțiuni posibile: deny-packet, reset-tcp-connection, etc.

! Dezactivare semnătură exemplu (ID 2004) 
R1(config-ips)# signature 2004 disable
R1(config-ips)# exit
```

### 2.3 Verificarea configurației IPS

```bash
R1# show ip ips configuration
R1# show ip ips statistics
R1# show ip ips interface
```
- Confirmați că *MYIPS* este asociat la interfața Fa0/0 in-bound.  
- Observați numărul de alerte, pachete inspectate etc.

### 2.4 Testarea funcționalității IPS

1. **Scanner** (Nmap/SuperScan) pe PC-A sau alt PC din rețea. Scanați IP-ul routerului și încercați să generați trafic suspect (ex. *OS detection*, *FIN scan*, etc.).  
2. **Observați** dacă routerul raportează evenimente de tip “IPS Alert” în consolă sau în `show ip ips statistics`.  

### 2.5 Redirecționarea mesajelor IPS către un server Syslog

```bash
R1(config)# logging host 192.168.1.100
R1(config)# logging trap informational
```
Mesajele IPS vor fi trimise către serverul Syslog. Verificați pe server dacă apar loguri.

---

## Partea 3: Configurarea IPS folosind SDM

Dacă doriți să configurați **IPS** pe alt router (ex. **R3**) sau pe același, puteți folosi Cisco SDM:

### 3.1 Accesarea SDM

1. **Pe R3** (sau R1, după caz), activați serverul HTTP/HTTPS:
   ```bash
   R3(config)# ip http server
   R3(config)# ip http secure-server
   ```
2. **PC**: Deschideți un browser, accesați `https://192.168.3.1` (după IP-ul interfeței routerului).  
3. **Autentificați-vă** (folosind user/enable password).  

### 3.2 Wizard IPS în SDM

1. În *Cisco SDM*, navigați la **Configure** → **Intrusion Prevention** → **IPS**.  
2. Alegeți **IPS Basic Setup** sau **IPS Wizard**.  
3. Specificați **locația semnăturilor** (TFTP/flash) și directorul local (`flash:/ips`).  
4. Configurați **interfața** pe care să aplicați IPS (similar cu CLI, puteți alege inbound/outbound).  
5. **Aplică** setările. SDM va genera comenzi IOS echivalente.

### 3.3 Personalizarea semnăturilor

În **SDM** puteți:  
- **Exclude** semnături (disable)  
- **Modifica** acțiuni (alert, drop, reset etc.)  
- **Schimba** severitatea threshold

### 3.4 Monitorizare și testare

1. În **SDM** → **Monitor** → **IPS Status**, urmăriți alertele.  
2. Lansați un *scan Nmap* de pe PC-C (sau PC-A) spre routerul cu IPS.  
3. Observați **rapoartele** de evenimente/alerte în timp real.

---
