## LAB 6: Securizarea comutatoarelor de nivel 2

## Obiective

### Partea 1: Configurați setările de bază ale comutatorului
- Construiți topologia conform cerințelor.
- Configurați numele de gazdă, adresa IP pentru management și parolele de acces.

### Partea 2: Configurați accesul SSH la comutatoare
- Configurați un server SSH pe fiecare switch (S1, S2).
- Configurați un client SSH (pe PC) și verificați conectarea la switch.
- Verificați autentificarea și funcționalitatea SSH.

### Partea 3: Securitate pentru trunkuri și porturi de acces
- Configurați porturile trunk (mod trunk).
- Schimbați VLAN-ul “native”/“autotron” (numit și VLAN de bază) pentru porturile trunk.
- Verificați configurația trunk-ului.
- Activați controlul furtunii (storm control) pentru broadcast/multicast/unicast.
- Configurați porturile în modul access.
- Activați protecția **PortFast** și **BPDU Guard** pe porturile access.
- Verificați paza BPDU (BPDU Guard).  
- Activați protecția rădăcină (Root Guard) unde e nevoie.
- Configurați securitatea portului (port security) pe porturile access.
- Verificați că securitatea portului funcționează.
- Dezactivați (shutdown) porturile neutilizate.

### Partea 4: Configurați SPAN și monitorizați traficul
- Configurați un port SPAN (monitor) pe comutator.
- Conectați un PC cu Wireshark la portul de monitorizare.
- Analizați traficul și observați evenimentele de rețea (de exemplu, un “flood” generat de un atac).

---

## Detalii și topologie generală

- **Echipamente**: 
  - 1 router (ex. Cisco 1841 cu IOS 12.4(20) T1 sau similar) – opțional, pentru interfațarea cu rețeaua externă.  
  - 2 comutatoare (ex. Cisco 2960, imagine LAN Base 12.2(46)SE sau similar) cu suport SSH.  
  - PC-uri (Windows, cu PuTTY SSH, Wireshark, SuperScan/Nmap).  
  - Cabluri Ethernet (ex. straight-through) și un cablu rollover pentru consola switchului.  

- **Notă**: Asigurați-vă că switch-urile (S1, S2) nu au configurații de pornire existente (comenzile `erase startup-config` și `reload`).

Un exemplu de topologie:

```
         [PC-A]                   [PC-B]
           |                         |
        (Fa0/1)                   (Fa0/2)
            \                       /
             \                     /
              S1 ---- trunk ---- S2
             /                     \
            /                       \
        (Fa0/3)                   (Fa0/4)
         [Router]                 (PC de monitorizare SPAN)
```

Adaptați la propriile voastre echipamente și scopuri.

---

## Partea 1: Configurare de bază a comutatorului

### 1.1 Conectarea prin consola switchului

1. **Cablare**: Folosiți cablul rollover de la PC (port serial/USB) la portul `Console` al switchului.  
2. **Parametrii terminalului**: `9600 baud, 8N1, no flow control`.  

### 1.2 Configurarea parametrilor de bază (exemplu pentru S1)

```bash
Switch> enable
Switch# configure terminal

! Setare nume (hostname)
Switch(config)# hostname S1

! Parola de enable
S1(config)# enable secret cisco123

! Linie de consolă
S1(config)# line console 0
S1(config-line)# password cisco
S1(config-line)# login
S1(config-line)# exit

! Linie VTY (folosită ulterior pentru SSH)
S1(config)# line vty 0 4
S1(config-line)# password cisco
S1(config-line)# login
S1(config-line)# exit

! (Opțional) Banner
S1(config)# banner motd #
Acces neautorizat interzis!
#

! Configurare IP pentru management (pe VLAN 1 sau alt VLAN dedicat)
S1(config)# interface vlan 1
S1(config-if)# ip address 192.168.1.2 255.255.255.0
S1(config-if)# no shutdown
S1(config-if)# exit

! Definire gateway (dacă avem un router)
S1(config)# ip default-gateway 192.168.1.1

! Activare criptare parole
S1(config)# service password-encryption

S1(config)# end
S1# copy running-config startup-config
```

**Repetați** pașii similari pentru S2 cu adrese IP corespunzătoare (ex. `192.168.1.3/24`).

---

## Partea 2: Configurați accesul SSH la comutatoare

### 2.1 Configurări SSH pe switch

1. **Nume de domeniu** (obligatoriu pentru generarea cheilor RSA)

```bash
S1(config)# ip domain-name cisco.local
```

2. **Generarea cheilor RSA**:

```bash
S1(config)# crypto key generate rsa
How many bits in the modulus [512]: 1024
```

3. **Crearea unui user local**:

```bash
S1(config)# username admin privilege 15 secret admin123
```

4. **Activarea autentificării SSH pe VTY**:

```bash
S1(config)# line vty 0 4
S1(config-line)# transport input ssh
S1(config-line)# login local
S1(config-line)# exit
```

5. **(Opțional) Dezactivare telnet**:

```bash
S1(config)# line vty 0 4
S1(config-line)# transport input ssh
S1(config-line)# exit
```

### 2.2 Testare SSH

1. **Pe PC** (Windows) – deschideți `PuTTY`.  
2. Conectați-vă la IP-ul management al switchului (ex. `192.168.1.2`) pe portul 22 (SSH).  
3. Introduceți userul `admin` și parola `admin123`.  
4. Verificați că puteți intra în modul exec privilegiat (`enable`) fără probleme.

---

## Partea 3: Securitate trunk și porturi de acces

### 3.1 Configurați modul trunk pe porturile care leagă S1 și S2

Presupunem că Fa0/24 pe S1 și Fa0/24 pe S2 sunt interfațele trunk. Pe **S1**:

```bash
S1# configure terminal
S1(config)# interface fa0/24
S1(config-if)# switchport mode trunk
S1(config-if)# switchport trunk encapsulation dot1q   ! Dacă este suportat
S1(config-if)# switchport trunk native vlan 99        ! Schimbăm VLAN nativ în 99 (de ex.)
S1(config-if)# no shutdown
S1(config-if)# end
```
Pe **S2** repetați aceiași pași (portul relevant, fa0/24).

### 3.2 Verificați configurația trunk

```bash
S1# show interfaces trunk
```
Ar trebui să vedeți portul Fa0/24 în modul trunk cu VLAN nativ 99.

### 3.3 Activați controlul furtunii (storm control)

Exemplu, pentru **broadcast** pe porturile acces (ex. Fa0/1, Fa0/2…):

```bash
S1(config)# interface range fa0/1 - 12
S1(config-if-range)# storm-control broadcast level 10.00
! Asta limitează broadcast la 10% din bandă
S1(config-if-range)# storm-control multicast level 10.00
S1(config-if-range)# end
```
Se poate face și pentru unicast, după nevoie.

### 3.4 Configurați porturile de acces

Presupunem că Fa0/1-12 sunt pentru PC-uri:

```bash
S1(config)# interface range fa0/1 - 12
S1(config-if-range)# switchport mode access
S1(config-if-range)# switchport access vlan 10
S1(config-if-range)# spanning-tree portfast
S1(config-if-range)# spanning-tree bpduguard enable
S1(config-if-range)# end
```
- **PortFast**: reduce timpii de convergență STP pentru stații finale.  
- **BPDU Guard**: dezactivează portul dacă primește BPDUs, prevenind bucle formate de un comutator neautorizat.

### 3.5 Verificați protecția BPDU

```bash
S1# show running-config | include bpduguard
```
Ar trebui să vedeți `spanning-tree bpduguard enable`.

### 3.6 Activați protecția rădăcină (Root Guard)

Pe porturile trunk unde **S1** este considerat “root” și nu vrei alt switch să preia rolul de root:

```bash
S1(config)# interface fa0/24
S1(config-if)# spanning-tree guard root
S1(config-if)# end
```

### 3.7 Configurarea securității portului (port security)

Exemplu, pentru porturile acces (Fa0/1-12):

```bash
S1(config)# interface range fa0/1 - 12
S1(config-if-range)# switchport port-security
S1(config-if-range)# switchport port-security maximum 2
! Permitem max. 2 adrese MAC
S1(config-if-range)# switchport port-security violation shutdown
! Când se încalcă regula, portul e oprit
S1(config-if-range)# switchport port-security mac-address sticky
! MAC-urile învățate se salvează automat în running-config
S1(config-if-range)# end
```

### 3.8 Verificați securitatea portului

```bash
S1# show port-security interface fa0/1
```
Veți vedea starea, numărul de MAC-uri, încălcări etc.

### 3.9 Dezactivați porturile neutilizate

Dacă Fa0/13-23 nu sunt folosite, le puneți în VLAN “disconnected” și le dați `shutdown`:

```bash
S1(config)# interface range fa0/13 - 23
S1(config-if-range)# switchport mode access
S1(config-if-range)# switchport access vlan 999
S1(config-if-range)# shutdown
S1(config-if-range)# end
```
(Asigurați-vă că VLAN 999 există creat, chiar dacă nu e routat.)

---

## Partea 4: Configurați SPAN și monitorizați traficul

### 4.1 Configurare SPAN

1. **Identificați** portul sursă (cel pe care doriți să monitorizați).  
2. **Identificați** portul destinație (cel conectat la PC cu Wireshark).

Exemplu, monitorizați tot traficul de pe Fa0/1 și trimiteți-l spre Fa0/2:

```bash
S1(config)# monitor session 1 source interface fa0/1
S1(config)# monitor session 1 destination interface fa0/2
```

> În mod real, portul Fa0/2 trebuie să fie conectat la PC cu Wireshark, iar acel PC să nu aibă alt trafic de rețea.

### 4.2 Lansați Wireshark

1. **Pe PC** conectat la portul Fa0/2 (destinație SPAN), deschideți Wireshark.  
2. Selectați interfața de rețea locală.  
3. Ar trebui să vedeți pachete provenite de pe Fa0/1.

### 4.3 Generați trafic și observați-l

1. **De pe PC-A** (conectat la Fa0/1) faceți ping/telnet/ssh către alte adrese.  
2. **În Wireshark**, observați pachetele care apar.  
3. (Opțional) Lansați un atac/scan (de ex. cu *SuperScan*) și observați fluxul de trafic în Wireshark.

---

## Verificări finale

1. **`show interfaces trunk`** – asigurați-vă că trunkurile sunt corect configurate și VLAN nativ nu e VLAN 1.  
2. **`show spanning-tree`** – verificați starea STP, portfast, root guard, etc.  
3. **`show storm-control`** – vedeți parametrii broadcast/multicast/unicast.  
4. **`show port-security interface fa0/1`** – confirmă starea port security.  
5. **`show monitor session 1`** – confirmă configurația SPAN.  
6. **Test**: SSH la switch, ping între PC-uri, monitorizare trafic SPAN în Wireshark.  

---
