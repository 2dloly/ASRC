## LAB 3: Asigurarea accesului administrativ folosind AAA și RADIUS

### Obiective

1. **Partea 1**: Configurare de bază a dispozitivului de rețea  
   - Configurați setările de bază (nume, adrese IP, parole de acces)  
   - Configurați rutarea statică  

2. **Partea 2**: Configurați autentificare locală  
   - Creați un utilizator de bază de date locală și accesați local pentru liniile `console`, `vty` și `aux`  
   - Testați configurația  

3. **Partea 3**: Configurarea autentificării locale folosind AAA  
   - Configurați baza de date locală a utilizatorilor prin Cisco IOS  
   - Configurați autentificarea locală AAA folosind Cisco IOS  
   - Configurați autentificarea locală AAA folosind SDM (opțional)  
   - Testați configurația  

4. **Partea 4**: Configurarea autentificării centralizate folosind AAA și RADIUS  
   - Instalați un server RADIUS pe un computer  
   - Configurați utilizatorii pe serverul RADIUS  
   - Configurați serviciile AAA pe router pentru a accesa serverul RADIUS (CLI și/sau SDM)  
   - Testați autentificarea AAA RADIUS  

---

## Resurse necesare

- **3 routere** cu **SDM 2.5 instalat** (Cisco 1841 cu Cisco IOS Release 12.4 (20) T1 sau compatibil)  
- **2 comutatoare** (Cisco 2960 sau comparabile)  
- **PC-A**: Windows XP, Vista sau Windows Server cu software de terminal (PuTTY) + opțional software RADIUS (dacă îl folosiți pe aceeași mașină)  
- **PC-C**: Windows XP sau Vista + terminal (pentru testare)  
- **Cabluri Serial și Ethernet** conform topologiei din laborator  
- **Cabluri rollover** pentru conectare prin portul de consolă  

> **Notă**: Asigurați-vă că routerele și comutatoarele nu au configurații de pornire (`erase startup-config`).  

---

## Partea 1: Configurare de bază a dispozitivului de rețea

### 1.1 Conectarea fizică și accesul la consola routerului

1. **Cablare**: Conectați routerele între ele și la switch-uri conform schemei.  
2. **Port de consolă**: Utilizați cablul **rollover** din portul serial sau USB (cu adaptor) al PC-ului către portul `Console` al routerului.  
3. **Terminal**: Deschideți PuTTY (sau alt client), setați `9600 baud`, `8 data bits`, `No parity`, `1 stop bit`, `No flow control`.  

### 1.2 Configurarea inițială (hostname, parole, interfețe)

Pe exemplul routerului **R1**:

```bash
Router> enable
Router# configure terminal
Router(config)# hostname R1

! Configurare parolă pentru modul EXEC privilegiat
R1(config)# enable secret cisco123

! Configurare linie de consolă
R1(config)# line console 0
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! (opțional) Configurare linie AUX
R1(config)# line aux 0
R1(config-line)# password cisco
R1(config-line)# login
R1(config-line)# exit

! Configurare interfețe (exemplu):
R1(config)# interface fastEthernet 0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

! (opțional) Configurare interfață serială
R1(config)# interface serial 0/0/0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# clock rate 64000   ! numai pe capătul DCE
R1(config-if)# no shutdown
R1(config-if)# exit

! Activare criptare parole
R1(config)# service password-encryption

! Banner
R1(config)# banner motd #
ATENȚIE: Acces restricționat. Conectarea neautorizată este interzisă!
#

R1(config)# end
R1# copy running-config startup-config
```

> Repetați pașii similari pentru R2 și R3, cu adrese IP și parole potrivite topologiei voastre.

### 1.3 Rutare statică

Definiți rute statice între rețele, în funcție de schema IP. De exemplu, dacă R1 are rețeaua `192.168.1.0/24` și conectare WAN către R2 pe `10.10.10.0/30`:

```bash
R1(config)# ip route 192.168.2.0 255.255.255.0 10.10.10.2
```

> Ajustați pentru fiecare router, astfel încât orice PC de la un capăt să poată comunica cu oricare altă rețea.

---

## Partea 2: Configurați autentificare locală

Înainte să implementăm AAA, putem defini autentificarea locală în modul clasic (fără `aaa new-model`).

### 2.1 Creare utilizator local pentru liniile console, aux, vty

```bash
R1# configure terminal

! Creăm un utilizator local:
R1(config)# username localUser secret localPass

! Atribuim acest utilizator pentru conectare pe VTY:
R1(config)# line vty 0 4
R1(config-line)# login local
R1(config-line)# transport input telnet ssh
R1(config-line)# exit

! (opțional) Pentru consolă, dacă doriți același user (în loc de parola console):
R1(config)# line console 0
R1(config-line)# login local
R1(config-line)# exit

R1(config)# end
R1# copy running-config startup-config
```

### 2.2 Testare

1. **Telnet/SSH** către router (`telnet 192.168.1.1` sau `ssh localUser@192.168.1.1`), introduceți `localPass`.  
2. Verificați că puteți accesa modul exec și **enable**. (Dacă enable secret rămâne tot `cisco123`, trebuie introdus la trecerea în nivel privilegiat.)

---

## Partea 3: Configurarea autentificării locale folosind AAA

Acum, trecem la modul AAA pe router. Acest lucru ne permite un control mai detaliat al autentificării, autorizării și contabilizării (logging al acțiunilor).

### 3.1 Activarea modului AAA

```bash
R1# configure terminal
R1(config)# aaa new-model
```

> Odată activat `aaa new-model`, configurația clasică (login local, password cisco pe linii) este înlocuită de politica AAA.

### 3.2 Configurarea bazei de date locale AAA

```bash
! Definim utilizatori în baza de date locală AAA
R1(config)# username admin secret admin123
R1(config)# username user1 secret user123
```

### 3.3 Crearea metodelor de autentificare AAA

```bash
! Metodă de autentificare pentru consola si vty:
R1(config)# aaa authentication login default local
```

- `login default` se aplică implicit tuturor liniilor (console, vty) pentru autentificare.  
- `local` înseamnă că se bazează pe userii definiți local (admin, user1, etc.).

### 3.4 Asocieri linii cu politicile AAA

```bash
R1(config)# line console 0
R1(config-line)# login authentication default
R1(config-line)# exit

R1(config)# line vty 0 4
R1(config-line)# login authentication default
R1(config-line)# transport input ssh telnet
R1(config-line)# exit
```

### 3.5 (Opțional) Configurare autorizare AAA

Dacă dorim să controlăm comenzile permise unui user (de exemplu user1 să nu aibă acces la modul config), putem folosi:

```bash
R1(config)# aaa authorization exec default local 
```

Acest lucru va verifica ce nivel de privilege are userul. Un user cu `privilege 15` va intra direct în modul exec privilegiat.

### 3.6 Testare

1. **Conectați-vă** prin consolă sau SSH.  
2. **Introduceți** userul (`admin` / `user1`) și parola.  
3. **Verificați** ce nivel de acces aveți (`show privilege`).  
4. **Intrați** în `enable`: se folosește parola `enable secret cisco123` dacă nu ați configurat altfel.  

---

## Partea 4: Configurarea autentificării centralizate folosind AAA și RADIUS

Pentru a folosi un server RADIUS:

1. Instalați software RADIUS pe un PC (ex. *FreeRADIUS* pe Linux sau *NPS* pe Windows Server).  
2. Creați utilizatori și parole în baza de date RADIUS.  
3. Configurați routerul să folosească serverul RADIUS pentru autentificare.

### 4.1 Configurarea serverului RADIUS (exemplu generic)

- **IP server**: `192.168.1.100`  
- **Cheia partajată (shared secret)**: `radiusSecret`  
- **Utilizatori**:  
  - `radiusAdmin / pass123`  
  - `radiusUser / pass456`

Configurați serverul RADIUS conform documentației software.

### 4.2 Configurarea AAA pe router pentru RADIUS

```bash
R1# configure terminal
R1(config)# aaa new-model

! Definiți serverul RADIUS
R1(config)# radius server RADIUS-SRV
R1(config-radius-server)# address ipv4 192.168.1.100 auth-port 1812 acct-port 1813
R1(config-radius-server)# key radiusSecret
R1(config-radius-server)# exit

! Metoda de autentificare la login: mai întâi RADIUS, apoi fallback local
R1(config)# aaa authentication login default group radius local

! (Opțional) Metoda de autentificare la enable: RADIUS, fallback local
R1(config)# aaa authentication enable default group radius enable

! Metoda de autorizare (opțională, dacă doriți control asupra comenzilor)
R1(config)# aaa authorization exec default group radius local
```

> Sintaxa poate varia puțin în funcție de versiunea de IOS. Pentru IOS mai vechi, se folosește `aaa group server radius RADIUS-GRP` și `server x.x.x.x`, `key ...`, etc.

### 4.3 Aplicare politică AAA pe linii

```bash
R1(config)# line vty 0 4
R1(config-line)# login authentication default
R1(config-line)# transport input ssh telnet
R1(config-line)# exit

! idem line console 0, dacă doriți la consolă să folosească AAA
R1(config)# line console 0
R1(config-line)# login authentication default
R1(config-line)# exit
```

### 4.4 Testare cu RADIUS

1. **Dintr-un PC** (ex. PC-A), faceți `ssh` sau `telnet` către R1.  
2. **Introduceți** userul definit pe serverul RADIUS (ex. `radiusUser` / `pass456`).  
3. **Observați** că routerul consultă serverul RADIUS pentru validare.  
4. **Verificați** pe serverul RADIUS logul de acces.  
---
