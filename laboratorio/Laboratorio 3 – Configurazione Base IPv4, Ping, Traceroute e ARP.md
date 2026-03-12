
> **Corso:** Reti di Calcolatori | **Strumento:** Kathara | **Data:** 2 marzo 2026

---
## 1. Topologia di rete

Il laboratorio simula una rete con **2 router** (r1, r2) e **3 host** (pc1, pc2, pc3) interconnessi tramite **3 LAN** (A, B, C).

```
                   LAN A (195.11.14.0/24)
        ┌──────────────────────────────┐
        │                              │
      pc1 (.5)                      r1 (eth0: .1)
                                    r1 (eth1: .9)
                                        │
                   LAN B (100.0.0.8/30) │
                                    r2 (eth1: .10)
                                    r2 (eth0: .1)
                                        │
                   LAN C (200.1.1.0/24) │
        ┌────────────────────────────────┐
        │             │                  │
      pc2 (.7)      pc3 (.3)           r2 (.1)
```

**Osservazione del docente:** "Abbiamo tre domini di collisione divisi da due router. La LAN B interconnette semplicemente i due router – è una rete punto-punto."

---

## 2. Piano di indirizzamento IP e MAC

### Indirizzi IP

|Dispositivo|Interfaccia|Indirizzo IP|Prefisso|LAN|
|---|---|---|---|---|
|pc1|eth0|195.11.14.5|/24|A|
|r1|eth0|195.11.14.1|/24|A|
|r1|eth1|100.0.0.9|/30|B|
|r2|eth1|100.0.0.10|/30|B|
|r2|eth0|200.1.1.1|/24|C|
|pc2|eth0|200.1.1.7|/24|C|
|pc3|eth0|200.1.1.3|/24|C|

### Indirizzi MAC (semplificati)

|Dispositivo|Interfaccia|MAC Address|
|---|---|---|
|pc1|eth0|00:00:00:00:00:01|
|pc2|eth0|00:00:00:00:00:02|
|pc3|eth0|00:00:00:00:00:03|
|r1|eth0 (LAN A)|00:00:00:00:00:a1|
|r1|eth1 (LAN B)|00:00:00:00:00:b1|
|r2|eth1 (LAN B)|00:00:00:00:00:b2|
|r2|eth0 (LAN C)|00:00:00:00:00:c1|

> **💡 Trucco mnemonico:** La lettera nel MAC identifica la LAN (a1 = LAN A, b1/b2 = LAN B, c1 = LAN C).

### La LAN B – prefisso /30 spiegato

La LAN B ha prefisso `100.0.0.8/30` – una subnet particolarmente compatta.

```
Ultimo byte in binario:  0 0 0 0 1 0 0 0
                         ─────────────┘└─
                         bit di rete    bit host (solo 2 bit liberi)

Bit di rete fissi: 000010  → valore decimale 8
Indirizzi host disponibili:
  00 → indirizzo di rete (100.0.0.8)   ← non usabile
  01 → 100.0.0.9   → assegnato a r1
  10 → 100.0.0.10  → assegnato a r2
  11 → broadcast (100.0.0.11)           ← non usabile
```

**Domanda tipica d'esame:** _Quanti indirizzi utilizzabili ha una /30?_ → **2 soli indirizzi** host (2² - 2 = 2). Perfetto per link punto-punto tra router.

---

## 3. File di configurazione del laboratorio

### `lab.conf` – topologia e MAC

```bash
r1[0]="A/00:00:00:00:00:a1"   # r1.eth0 → LAN A
r1[1]="B/00:00:00:00:00:b1"   # r1.eth1 → LAN B
r1[image]="kathara/base"
r1[ipv6]="false"

r2[0]="C/00:00:00:00:00:c1"   # r2.eth0 → LAN C
r2[1]="B/00:00:00:00:00:b2"   # r2.eth1 → LAN B
r2[image]="kathara/base"
r2[ipv6]="false"

pc1[0]="A/00:00:00:00:00:01"  # pc1.eth0 → LAN A
pc1[image]="kathara/base"
pc1[ipv6]="false"

pc2[0]="C/00:00:00:00:00:02"  # pc2.eth0 → LAN C
pc2[image]="kathara/base"
pc2[ipv6]="false"

pc3[0]="C/00:00:00:00:00:03"  # pc3.eth0 → LAN C
pc3[image]="kathara/base"
pc3[ipv6]="false"

wireshark[bridged]=true
wireshark[port]="3000:3000"
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[num_terms]=0
```

### File `.startup` – configurazione IP degli host

**`pc1.startup`**

```bash
ip address add 195.11.14.5/24 dev eth0
ip route add default via 195.11.14.1         # default gateway = r1
```

**`pc2.startup`**

```bash
ip address add 200.1.1.7/24 dev eth0
ip route add default via 200.1.1.1 dev eth0  # default gateway = r2
```

**`pc3.startup`**

```bash
ip address add 200.1.1.3/24 dev eth0
ip route add default via 200.1.1.1 dev eth0  # default gateway = r2
```

**`r1.startup`**

```bash
ip address add 195.11.14.1/24 dev eth0       # interfaccia LAN A
ip address add 100.0.0.9/30 dev eth1         # interfaccia LAN B
ip route add 200.1.1.0/24 via 100.0.0.10 dev eth1  # rotta verso LAN C
```

**`r2.startup`**

```bash
ip address add 200.1.1.1/24 dev eth0         # interfaccia LAN C
ip address add 100.0.0.10/30 dev eth1        # interfaccia LAN B
ip route add 195.11.14.0/24 via 100.0.0.9 dev eth1 # rotta verso LAN A
```

> **Logica di configurazione (spiegazione del docente):** "Un router conosce automaticamente le LAN direttamente connesse. Quello che bisogna dirgli esplicitamente è come raggiungere le LAN remote. R1 vede A e B direttamente – deve solo sapere come arrivare a C, e lo fa attraverso R2 (next hop: 100.0.0.10)."

---

## 4. Comandi fondamentali

### Avvio del laboratorio

```bash
cd kathara-lab_basic-ipv4
kathara lstart
```

### Controllare indirizzi IP di un'interfaccia

```bash
ip address
```

**Output esempio su pc1:**

```
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo
7: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 195.11.14.5/24 scope global eth0
```

> **Nota:** L'interfaccia `lo` (loopback) con indirizzo `127.0.0.1/8` è presente su **tutti** i dispositivi. Rappresenta la macchina stessa e serve per comunicazioni interne allo stack protocollare.

### Visualizzare la tabella di routing

```bash
routel
```

**Output esempio su pc1:**

```
Dst              Gateway          Prefsrc  Protocol  Scope  Dev   Table
default          195.11.14.1                                 eth0
195.11.14.0/24                   195.11.14.5  kernel  link  eth0
127.0.0.0/8                      127.0.0.1    kernel  host  lo    local
127.0.0.1                        127.0.0.1    kernel  host  lo    local
195.11.14.5                      195.11.14.5  kernel  host  eth0  local
195.11.14.255                    195.11.14.5  kernel  link  eth0  local
```

**Come leggere l'output di `routel`:**

- Le righe con **prefisso esplicito** (es. `195.11.14.0/24`) sono le rotte principali
- La riga `default` è la rotta di default (invia tutto il traffico non locale al gateway)
- Le righe con **indirizzi singoli** (host, broadcast) sono aggiunte automaticamente dal kernel
- Rotte valide di pc1: `default` + `195.11.14.0/24` + loopback

> **Docente:** "Il default gateway deve avere un indirizzo IP che appartenga a una LAN **direttamente connessa** alla macchina. Altrimenti la riga sarebbe inutile!"

### Ispezionare la cache ARP

```bash
arp -n        # mostra la cache ARP senza risolvere DNS
```

**Output esempio su pc3 (dopo un ping):**

```
Address          HWtype  HWaddress           Flags Mask  Iface
200.1.1.7        ether   00:00:00:00:00:02   C           eth0
```

### Collegare Wireshark a un dominio di collisione

```bash
# Dall'host, nella directory del lab:
kathara lconfig -n wireshark --add C    # attacca wireshark alla LAN C
kathara lconfig -n wireshark --add B    # attacca wireshark alla LAN B
```

Poi aprire il browser su `localhost:3000` e selezionare l'interfaccia `eth1`, `eth2`, ecc.

### Ping

```bash
ping <indirizzo_IP>
```

### Traceroute

```bash
traceroute <indirizzo_IP> -z 1    # -z 1: intervallo minimo tra sonde (utile per evitare problemi)
```

---

## 5. Esperienza 1 – ARP: ping da pc3 a pc2 (traffico locale)

**Scenario:** pc3 → pc2 (entrambi su LAN C). Wireshark collegato alla LAN C.

### Sequenza osservata

**Passo 1: ARP cache vuota prima del ping**

```bash
root@pc3:/# arp -n
# (nessun output – cache vuota)
```

**Passo 2: Ping**

```bash
root@pc3:/# ping 200.1.1.7
64 bytes from 200.1.1.7: icmp_seq=1 ttl=64 time=1.93 ms
64 bytes from 200.1.1.7: icmp_seq=2 ttl=64 time=0.638 ms
```

**Passo 3: ARP cache dopo il ping**

```bash
root@pc3:/# arp -n
Address     HWtype  HWaddress           Flags  Iface
200.1.1.7   ether   00:00:00:00:00:02   C      eth0
```

**Passo 4: ARP cache anche su pc2 (RFC 826 – comportamento standard)**

```bash
root@pc2:/# arp -n
Address     HWtype  HWaddress           Flags  Iface
200.1.1.3   ether   00:00:00:00:00:03   C      eth0
```

### Cosa ha visto Wireshark

```
1. ARP Request  (broadcast): "Chi ha 200.1.1.7? Risponda 200.1.1.3"
2. ARP Reply    (unicast):   "200.1.1.7 è 00:00:00:00:00:02"
3. ICMP Echo Request  pc3 → pc2
4. ICMP Echo Reply    pc2 → pc3
...
(al termine del ping:)
5. ARP unicast query: "Sei ancora raggiungibile?"  ← dialogo di mantenimento cache
6. ARP unicast reply
```

### Perché i pacchetti non passano per i router?

**Il traffico interno alla stessa rete locale non attraversa mai i router.**

pc3 controlla se `200.1.1.7` appartiene alla sua stessa rete (`200.1.1.0/24`). Sì → invia direttamente. Fa una ARP request per trovare il MAC di pc2 e comunica direttamente con lui via Ethernet.

---

## 6. Esperienza 2 – ARP: ping da pc2 a pc1 (traffico non locale)

**Scenario:** pc2 → pc1 (LAN C → LAN A, attraverso r2 e r1). Wireshark collegato alla LAN B.

### Ping

```bash
root@pc2:/# ping 195.11.14.5
64 bytes from 195.11.14.5: icmp_seq=1 ttl=62 time=5.86 ms
64 bytes from 195.11.14.5: icmp_seq=2 ttl=64 time=1.69 ms
```

> **Nota:** `ttl=62` = TTL originale (64) − 2 hop (r2 e r1). Questo conferma il percorso attraverso due router.

### ARP cache di pc2 dopo il ping

```bash
root@pc2:/# arp -n
Address     HWtype  HWaddress           Flags  Iface
200.1.1.1   ether   00:00:00:00:00:c1   C      eth0   ← MAC di r2 (eth0)
200.1.1.3   ether   00:00:00:00:00:03   C      eth0
```

> **Comportamento fondamentale:** pc2 deve inviare il pacchetto fuori dalla sua LAN. Non può fare ARP verso pc1 (che si trova su un'altra LAN). Quindi fa ARP verso il suo **default gateway** (r2), e nella cache appare il MAC dell'interfaccia eth0 di r2, non quello di pc1!

### ARP cache di r1 e r2

```bash
root@r1:/# arp -n
Address      HWtype  HWaddress           Iface
195.11.14.5  ether   00:00:00:00:00:01   eth0   ← pc1
100.0.0.10   ether   00:00:00:00:00:b2   eth1   ← r2 (eth1)

root@r2:/# arp -n
Address      HWtype  HWaddress           Iface
100.0.0.9    ether   00:00:00:00:00:b1   eth1   ← r1 (eth1)
200.1.1.7    ether   00:00:00:00:00:02   eth0   ← pc2
```

> **Docente:** "Anche i router eseguono ARP ogni volta che devono inviare pacchetti su una LAN Ethernet. I router hanno le loro ARP cache!"

### Flusso completo del pacchetto (pc2 → pc1)

```
pc2 capisce: 195.11.14.5 NON è nella mia LAN → mando al gateway (r2)

[LAN C]  pc2 ---ARP?-mac-di-r2--→ r2.eth0
         pc2 ---ICMP Echo req--→  r2.eth0
                                  (IP dst: 195.11.14.5, MAC dst: MAC-r2)

[LAN B]  r2.eth1 ---ARP?-mac-di-r1--→ r1.eth1
         r2.eth1 ---ICMP Echo req--→  r1.eth1
                                      (IP dst: 195.11.14.5, MAC dst: MAC-r1)

[LAN A]  r1.eth0 ---ARP?-mac-di-pc1--→ pc1
         r1.eth0 ---ICMP Echo req--→   pc1
```

**Regola chiave:** L'indirizzo IP di destinazione **non cambia mai** lungo il percorso. Cambiano solo i MAC address ad ogni hop.

---

## 7. Esperienza 3 – Traceroute da pc2 a pc1

**Scenario:** Traceroute da pc2 verso pc1. Wireshark sulla LAN B (eth1).

### Esecuzione

```bash
root@pc2:/# traceroute 195.11.14.5 -z 1
traceroute to 195.11.14.5 (195.11.14.5), 30 hops max, 60 byte packets
 1  200.1.1.1  (200.1.1.1)   0.882 ms  0.662 ms  0.456 ms   ← r2 (eth0 su LAN C)
 2  100.0.0.9  (100.0.0.9)   0.903 ms  0.877 ms  1.218 ms   ← r1 (eth1 su LAN B)
 3  195.11.14.5 (195.11.14.5) 0.987 ms  1.354 ms  1.015 ms  ← pc1 (destinazione)
```

### Come funziona Traceroute

Traceroute usa **pacchetti UDP** con TTL crescente:

|Sonda|TTL|Raggiunge|Risposta ricevuta|
|---|---|---|---|
|1ª|1|r2|ICMP **Time Exceeded** da r2|
|2ª|2|r1|ICMP **Time Exceeded** da r1|
|3ª|3|pc1|ICMP **Port Unreachable** (destinazione raggiunta!)|

Per ogni TTL vengono inviate **3 sonde** (per misurare il tempo con più campioni).

### Cosa ha visto Wireshark sulla LAN B

```
TTL=1: [su LAN C: UDP inviato, r2 scarta e risponde Time Exceeded]
TTL=2: [su LAN B: UDP inoltrato da r2, r1 scarta e risponde Time Exceeded]
TTL=3: [su LAN B e A: UDP inoltrato, pc1 riceve → risponde Port Unreachable]
+ query ARP unicast intercalate durante il dialogo
```

### Parametro `-z 1`

`-z 1` specifica l'**intervallo minimo in millisecondi tra le sonde** (default 0). Usato per evitare problemi con alcune implementazioni di rete.

---

## 8. Concetti teorici chiave

### ARP – Address Resolution Protocol

**Scopo:** Trovare il MAC address corrispondente a un indirizzo IP all'interno della stessa LAN.

**Comportamento:**

- Prima di inviare, il mittente controlla la sua **ARP cache**
- Se non trova il MAC → invia una **ARP Request** in broadcast
- Chi riconosce il proprio IP risponde con una **ARP Reply** in unicast
- Il mittente salva la coppia IP↔MAC nella cache (RFC 826)
- Il **ricevitore** della ARP Request aggiorna anche la propria cache con il MAC del mittente (ottimizzazione RFC 826)

```
ARP Request (broadcast):  "Chi ha IP X? Risponda MAC-mio"
ARP Reply   (unicast):    "IP X ce l'ho io, il mio MAC è Y"
```

### Regola fondamentale IP vs. MAC

|Livello|Indirizzo|Cambia lungo il percorso?|
|---|---|---|
|Layer 3 (IP)|IP sorgente/destinazione|**No** – rimane fisso end-to-end|
|Layer 2 (Ethernet)|MAC sorgente/destinazione|**Sì** – cambia ad ogni hop|

### Tabella di routing – logica di costruzione

Per ogni nodo (host o router) le rotte si dividono in:

1. **Rotte direttamente connesse** – aggiunte automaticamente dal kernel quando si assegna un indirizzo IP a un'interfaccia
2. **Rotta di default** – per gli host, configurata manualmente (`ip route add default via <gateway>`)
3. **Rotte statiche esplicite** – per i router, verso LAN non direttamente connesse (`ip route add <rete> via <next-hop>`)

**Regola del gateway:** Il next-hop di qualsiasi rotta deve essere raggiungibile su una LAN **direttamente connessa**.

### Interfaccia di loopback

- Presente su ogni dispositivo, sempre con `127.0.0.1/8`
- Usata per comunicazioni interne (tra processi sulla stessa macchina)
- Non corrisponde a hardware fisico

### TTL e traceroute

- **TTL (Time To Live):** campo dell'header IP decrementato di 1 da ogni router
- Quando TTL = 0, il router scarta il pacchetto e invia **ICMP Time Exceeded** al mittente
- Traceroute sfrutta questo meccanismo aumentando il TTL progressivamente

---

## 9. Domande tipiche d'esame con risposte

**D: Cosa contiene la ARP cache di pc2 dopo un ping verso pc1 (LAN diversa)?** → Contiene il MAC del **default gateway** (r2), NON il MAC di pc1. Le ARP request non attraversano i router.

**D: Quanti indirizzi host ha una subnet /30? Perché si usa per link punto-punto?** → 2 indirizzi utilizzabili (2² - 2). Si usa per link tra due soli router, senza sprecare indirizzi.

**D: Il TTL del ping da pc2 a pc1 è 62, non 64. Perché?** → Il pacchetto ha attraversato 2 router (r2 e r1), ognuno ha decrementato il TTL di 1.

**D: Perché nella tabella di routing di r1 non c'è una rotta verso 195.11.14.0/24?** → È una rete direttamente connessa (r1 ha eth0 su quella rete) – il kernel la aggiunge automaticamente.

**D: Cosa succede se si fa ping verso un indirizzo inesistente nella LAN locale?** → Viene inviata una ARP Request in broadcast, nessuno risponde → "Destination Host Unreachable".

**D: Cosa succede se si fa ping verso un indirizzo inesistente fuori dalla LAN?** → Il pacchetto viene inviato al gateway, che risponde con "ICMP Network Unreachable" o "Host Unreachable" se non ha la rotta.

**D: Perché traceroute usa UDP e non ICMP come ping?** → Traceroute (implementazione Linux) usa UDP su porte alte (> 33434) senza applicazione in ascolto, così la destinazione risponde con "Port Unreachable" segnalando il raggiungimento. Alcuni traceroute usano ICMP Echo.

**D: Un host ha bisogno di una rotta di default per comunicare nella sua stessa LAN?** → No. La rotta verso la LAN locale è aggiunta automaticamente. La default route serve solo per il traffico **fuori** dalla LAN.

**D: In Wireshark sulla LAN C, cosa si vede quando pc2 fa ping a pc1?** → ARP request/reply per il MAC di r2 + pacchetti ICMP con MAC destinazione = MAC di r2 (non di pc1).

**D: Il comando `routel` su pc2 mostra molte righe. Quali sono le due rotte "vere"?** → La **rotta di default** (verso il gateway) e la **rotta della LAN locale** (200.1.1.0/24). Le altre righe sono indirizzi di host e broadcast aggiunti automaticamente dal kernel.

---

## Appendice – Riferimento rapido comandi

```bash
# Avvio lab
kathara lstart

# Controllare configurazione IP
ip address

# Visualizzare tabella di routing
routel

# Controllare/svuotare cache ARP
arp -n                           # mostra cache
arp -d <indirizzo>               # rimuove voce

# Ping
ping <IP>

# Traceroute
traceroute <IP> -z 1

# Collegare Wireshark a una LAN (dall'host, nella dir del lab)
kathara lconfig -n wireshark --add <LAN>   # es. --add C, --add B

# Aggiungere indirizzo IP a interfaccia
ip address add <IP>/<prefisso> dev <interfaccia>

# Aggiungere rotta
ip route add default via <gateway>
ip route add <rete>/<prefisso> via <next-hop> dev <interfaccia>
```