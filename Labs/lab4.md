## LAB 4: Configurarea firewall-urilor bazate pe CBAC și pe zonă

## Obiective

**Partea 1: Configurare de bază a routerului**  
- Configurați numele gazdelor, adresele IP ale interfețelor și parolele de acces.  
- Configurați **protocolul de rutare dinamic EIGRP**.  
- Folosiți un scanner (ex. **Nmap**) pentru a testa vulnerabilitățile routerului (porturi deschise, etc.).  

**Partea 2: Configurarea unui firewall de control de acces bazat pe context (CBAC)**  
- Configurați **CBAC** folosind **AutoSecure**.  
- Examinați configurația CBAC rezultată.  
- Verificați funcționarea firewall-ului (test ping/HTTP/alte servicii).  

**Partea 3: Configurarea unui firewall de politică bazat pe zonă (ZBF, ZPF sau ZFW)**  
- Configurați un firewall de politică bazat pe zonă manual sau folosind **SDM**.  
- Examinați configurația ZBF rezultată.  
- (Opțional) Folosiți **SDM Monitor** pentru a verifica configurația și testele de trafic.  

---

## Topologie și resurse necesare

- **3 x routere** (ex. Cisco 1841 cu IOS 12.4(20)T și SDM 2.5 instalat)  
- **2 x switch-uri** (ex. Cisco 2960)  
- **PC-A, PC-C** (Windows XP / Vista / 7) cu:
  - **PuTTY / SSH** pentru acces la console  
  - **Nmap** (sau alt scanner de porturi)  
  - **SDM** instalat (sau accesabil prin browser)  
- **Cabluri**: Serial & Ethernet conform topologiei și un **cablul rollover** pentru consola routerului.  

> **Notă**: Asigurați-vă că routerele și switch-urile nu au o configurație salvată înainte de a începe (comanda `erase startup-config`, `reload`).

---

## Partea 1: Configurare de bază a routerului

### 1.1 Setări inițiale și acces la consolă

1. **Cablare**: Conectați routerele între ele (WAN serial) și la switch-uri (LAN).  
2. **Conectare la consolă**:  
   - Folosiți un cablu **rollover** din portul de consolă al routerului la portul serial/USB al PC-ului.  
   - Deschideți PuTTY: 9600 baud, 8N1, no flow control.

### 1.2 Configurarea de bază (hostname, parole, interfețe)

Exemplu pe routerul **R1**:

```bash
Router> enable
Router# configure terminal
Router(config)# hostname R1

! Parola de enable (mod privilegiat)
R1(config)# enable secret cisco123

! Linie de consolă
R1(config)# line console 0
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! Linie VTY (pentru telnet/ssh)
R1(config)# line vty 0 4
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! Interfața LAN (ex. Fa0/0 -> 192.168.1.1/24)
R1(config)# interface fastEthernet 0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

! Interfața Serial -> R2 (ex. 10.10.10.1/30)
R1(config)# interface serial 0/0/0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# clock rate 64000     ! doar pe capătul DCE
R1(config-if)# no shutdown
R1(config-if)# exit

! Banner
R1(config)# banner motd #
ATENTIE: Acces Restrictionat!
#

! Criptarea parolelor
R1(config)# service password-encryption

R1(config)# end
R1# copy running-config startup-config
```

**Repetați** pașii similari pentru R2, R3.

### 1.3 Configurare EIGRP

**Pe R1** (exemplu: Autonomous System 1, rețele 192.168.1.0/24 și 10.10.10.0/30):

```bash
R1(config)# router eigrp 1
R1(config-router)# network 192.168.1.0 0.0.0.255
R1(config-router)# network 10.10.10.0 0.0.0.3
R1(config-router)# no auto-summary
R1(config-router)# end
```

**Pe R2** – repetați pentru rețelele direct conectate (de ex. 10.10.10.0/30, 10.10.20.0/30, etc.).  

**Pe R3** – repetați pentru rețelele de la capăt.  

Verificați cu:  
```bash
R1# show ip eigrp neighbors
R1# show ip route
```
Asigurați-vă că vedeți rutele.  

### 1.4 Testare conectivitate și Nmap

1. **Ping** de pe PC-A la gateway (R1 Fa0/0).  
2. **Ping** PC-A -> R2, R3.  
3. **Instalați** Nmap pe PC-A. Rulați:
   ```bash
   nmap -sS 192.168.1.1
   ```
   Aceasta scanează porturile deschise pe R1 (ex. Telnet/SSH, HTTP, etc.).  
4. Notați porturile deschise – veți observa cu firewall inactiv că majoritatea serviciilor implicite pot fi detectate.

---

## Partea 2: Configurarea unui firewall CBAC (Context-Based Access Control)

### 2.1 Concept

CBAC analizează traficul și permite doar conexiuni legitime. Pe scurt:
- Configurați liste de ACL de intrare minimă,  
- Activați `ip inspect` (CBAC) pe interfețe (în general, outbound-ul spre Internet),  
- CBAC creează automat reguli temporare pentru traficul de întoarcere (return traffic).

### 2.2 Configurarea CBAC prin AutoSecure

**AutoSecure** poate configura automat un firewall CBAC și dezactiva serviciile nefolosite.

```bash
R1# auto secure
```
Urmați pașii wizardului:
1. Va întreba dacă doriți să **dezactivați** serviciile nerestricționate.  
2. Vă va cere să configurați **CBAC** pe interfața percepută ca fiind “externă” (WAN) și să protejați “intern” (LAN).  
3. Va crea șabloane de inspecție pentru protocoale comune (TCP, UDP, ICMP).

### 2.3 Examinați configurația CBAC

După rularea AutoSecure, verificați cu:  
```bash
R1# show run | section ip inspect
R1# show run | section access-list
```
Observați:
- **`ip inspect name <cbac_name>`** cu protocoale: tcp, udp, icmp, etc.  
- ACL-urile generate de AutoSecure pe interfața externă (de obicei `inbound`) și `ip inspect <cbac_name> out` pe aceeași interfață.

### 2.4 Verificați funcționarea firewall-ului

1. **Ping** de pe PC-A către rețeaua externă (sau un server simulat pe R3).  
   - Traficul ar trebui să treacă, pentru că e inițiat din intern.  
2. **Scanner Nmap** din exterior (dacă R3 este considerat “extern”), scanați IP-ul R1.  
   - Ar trebui să vedeți mai puține porturi raportate deschise, datorită ACL-urilor.  
3. **Test trafic**: http, telnet etc. Observați că doar traficul permis de CBAC poate trece.

---

## Partea 3: Configurarea unui firewall de politică bazat pe zonă (ZBF)

### 3.1 Concept

- Se definesc **zone** (ex. `INSIDE`, `OUTSIDE`) și se asociază interfețele la aceste zone.  
- Se creează **zone-pair** (INSIDE->OUTSIDE, OUTSIDE->INSIDE) și se aplică o **politică** care definește ce trafic este inspectat, permis sau blocat.

### 3.2 Configurare manuală ZBF (CLI)

Exemplu simplificat pe **R1** cu interfața Fa0/0 (LAN) ca zonă INSIDE și Serial 0/0/0 (WAN) ca zonă OUTSIDE.

1. **Creați zone**:

```bash
R1(config)# zone security INSIDE
R1(config)# zone security OUTSIDE
```

2. **Asociați interfețele**:

```bash
R1(config)# interface fastEthernet 0/0
R1(config-if)# zone-member security INSIDE
R1(config-if)# exit

R1(config)# interface serial 0/0/0
R1(config-if)# zone-member security OUTSIDE
R1(config-if)# exit
```

3. **Configurați politicile de inspecție** (policy-map, class-map). Exemplu:

```bash
! Definire inspecție la nivel de protocol
R1(config)# policy-map type inspect default_policy
R1(config-pmap)# class type inspect class-default
R1(config-pmap-c)# inspect
R1(config-pmap-c)# exit
R1(config-pmap)# exit
```

4. **Creare zone-pair**:

```bash
R1(config)# zone-pair security INSIDE-TO-OUTSIDE source INSIDE destination OUTSIDE
R1(config-zone-pair)# service-policy type inspect default_policy
R1(config-zone-pair)# exit
```

5. **(Opțional)** Pentru sens invers (OUTSIDE->INSIDE), definiți alt zone-pair și o politică mai restrictivă (ex. doar trafic de retur).

```bash
R1(config)# zone-pair security OUTSIDE-TO-INSIDE source OUTSIDE destination INSIDE
R1(config-zone-pair)# service-policy type inspect default_policy
R1(config-zone-pair)# exit
```

6. **Verificare**:
   ```bash
   R1# show policy-map type inspect zone-pair
   R1# show zone security
   ```
   Observați că ZBF a creat un firewall stateful.  

### 3.3 Configurare ZBF folosind SDM

1. **Conectați-vă** la R1 dintr-un browser: `https://192.168.1.1` (dacă SDM este instalat și activat).  
2. **Autentificare** (user/enable pass).  
3. **Mergeți** la `Configure` → `Firewall and ACL`.  
4. **Alegeți** `Basic Firewall` or `Zone-Based Firewall Wizard`.  
   - SDM va cere ce interfață e “Inside” și ce interfață e “Outside”.  
   - Va propune reguli de inspecție pentru protocoale comune (ICMP, HTTP, Telnet, SSH).  
   - Aplicați și salvați configurația.  
5. **Verificare** în `Monitor` → `Firewall Status`.

---
