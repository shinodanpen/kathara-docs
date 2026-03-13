# Laboratorio 3 — Basic IPv4: ping, traceroute e ARP

← [09_Laboratori](09_Laboratori.md) | [01_INDEX](01_INDEX.md)

---

## Topologia

Due router (`r1`, `r2`) e tre host (`pc1`, `pc2`, `pc3`) collegati tramite tre LAN.
```
        pc1                          pc2   pc3
         │ eth0 (0:01)         (0:02)│     │(0:03)
         │                           │     │
    ─────┴──────── A ──────────── ───┴─────┴──── C
   195.11.14.0/24  │           200.1.1.0/24      │
               eth0│(0:a1)                   eth0│(0:c1)
                  r1                            r2
               eth1│(0:b1)                   eth1│(0:b2)
                   └──────── B ────────────────┘
                         100.0.0.8/30
```

### Prefissi e indirizzi IP

| Collision domain | Prefisso          | Device | Interfaccia | Indirizzo       |
|------------------|-------------------|--------|-------------|-----------------|
| A                | 195.11.14.0/24    | pc1    | eth0        | 195.11.14.5/24  |
| A                | 195.11.14.0/24    | r1     | eth0        | 195.11.14.1/24  |
| B                | 100.0.0.8/30      | r1     | eth1        | 100.0.0.9/30    |
| B                | 100.0.0.8/30      | r2     | eth1        | 100.0.0.10/30   |
| C                | 200.1.1.0/24      | r2     | eth0        | 200.1.1.1/24    |
| C                | 200.1.1.0/24      | pc2    | eth0        | 200.1.1.7/24    |
| C                | 200.1.1.0/24      | pc3    | eth0        | 200.1.1.3/24    |

### MAC address

I MAC address sono forzati in `lab.conf` per renderli facilmente leggibili durante l'analisi del traffico.

| Device | Interfaccia | MAC address         |
|--------|-------------|---------------------|
| pc1    | eth0        | 00:00:00:00:00:01   |
| pc2    | eth0        | 00:00:00:00:00:02   |
| pc3    | eth0        | 00:00:00:00:00:03   |
| r1     | eth0        | 00:00:00:00:00:a1   |
| r1     | eth1        | 00:00:00:00:00:b1   |
| r2     | eth0        | 00:00:00:00:00:c1   |
| r2     | eth1        | 00:00:00:00:00:b2   |

---

## File di configurazione

### `lab.conf`
```
r1[0]="A/00:00:00:00:00:a1"    # eth0 su LAN A con MAC forzato
r1[1]="B/00:00:00:00:00:b1"    # eth1 sul link punto-punto con r2
r1[image]="kathara/base"
r1[ipv6]="false"

r2[0]="C/00:00:00:00:00:c1"    # eth0 su LAN C con MAC forzato
r2[1]="B/00:00:00:00:00:b2"    # eth1 sul link punto-punto con r1
r2[image]="kathara/base"
r2[ipv6]="false"

pc1[0]="A/00:00:00:00:00:01"
pc1[image]="kathara/base"
pc1[ipv6]="false"

pc2[0]="C/00:00:00:00:00:02"
pc2[image]="kathara/base"
pc2[ipv6]="false"

pc3[0]="C/00:00:00:00:00:03"
pc3[image]="kathara/base"
pc3[ipv6]="false"

wireshark[bridged]=true             # necessario per raggiungere localhost:3000
wireshark[port]="3000:3000"
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[num_terms]=0
```

### `pc1.startup`
```bash
ip address add 195.11.14.5/24 dev eth0
ip route add default via 195.11.14.1    # default gateway = eth0 di r1 su LAN A
```

### `pc2.startup`
```bash
ip address add 200.1.1.7/24 dev eth0
ip route add default via 200.1.1.1 dev eth0    # default gateway = eth0 di r2 su LAN C
```

### `pc3.startup`
```bash
ip address add 200.1.1.3/24 dev eth0
ip route add default via 200.1.1.1 dev eth0    # stesso gateway di pc2, stessa LAN
```

### `r1.startup`
```bash
ip address add 195.11.14.1/24 dev eth0    # aggiunge implicitamente rotta directly connected per A
ip address add 100.0.0.9/30 dev eth1      # aggiunge implicitamente rotta directly connected per B
ip route add 200.1.1.0/24 via 100.0.0.10 dev eth1    # LAN C non è directly connected: next-hop = eth1 di r2
```

> Per abilitare il forwarding IP su r1, aggiungere `r1[sysctl]="net.ipv4.ip_forward=1"` in `lab.conf` oppure `sysctl -w net.ipv4.ip_forward=1` nel `.startup`. Vedi [abilitare il forwarding IP](07b_Router_Linux.md#abilitare-il-forwarding-ip).

### `r2.startup`
```bash
ip address add 200.1.1.1/24 dev eth0      # aggiunge implicitamente rotta directly connected per C
ip address add 100.0.0.10/30 dev eth1     # aggiunge implicitamente rotta directly connected per B
ip route add 195.11.14.0/24 via 100.0.0.9 dev eth1   # LAN A non è directly connected: next-hop = eth1 di r1
```

---

## Concetti chiave

### ARP cache

Ogni host e router mantiene una **ARP cache**: una tabella che mappa gli indirizzi IP ai MAC address appresi dinamicamente tramite il protocollo ARP. Le entry vengono aggiunte la prima volta che si comunica con un indirizzo e scadono dopo un timeout di inattività.
```bash
arp -n    # mostra la ARP cache senza risolvere gli IP via DNS
```

Output tipico:
```
Address     HWtype  HWaddress           Flags Mask  Iface
200.1.1.7   ether   00:00:00:00:00:02   C           eth0
```

| Colonna     | Descrizione                                             |
|-------------|---------------------------------------------------------|
| `Address`   | Indirizzo IP del vicino                                 |
| `HWtype`    | Tipo hardware (quasi sempre `ether`)                    |
| `HWaddress` | MAC address corrispondente                              |
| `Flags`     | `C` = entry appresa dinamicamente                       |
| `Iface`     | Interfaccia su cui è stata risolta                      |

Per i dettagli sul comportamento ARP negli host e nei router, vedi [ARP nei router](07b_Router_Linux.md#arp-nei-router).

### ARP per traffico locale vs. cross-router

Quando pc3 invia pacchetti a pc2 (stessa LAN C), l'ARP request viene risolta direttamente: la cache di pc3 conterrà il MAC di pc2, e il traffico non attraversa mai r2.

Quando pc2 invia pacchetti a pc1 (LAN diversa), l'indirizzo di destinazione è fuori dalla rete locale. pc2 non può emettere un'ARP request per 195.11.14.5 perché nessuno nella LAN C conosce quel MAC. Il kernel lo inoltra invece al default gateway (r2), e la cache di pc2 conterrà il MAC dell'interfaccia `eth0` di r2 — non quello di pc1.

### Traceroute e meccanismo TTL

`traceroute` scopre i router sul percorso sfruttando il campo **TTL** (Time To Live) dell'header IP. Invia una sequenza di probe UDP con TTL crescente a partire da 1:

- TTL=1: il primo router scarta il pacchetto e risponde con un messaggio ICMP **Time To Live exceeded**. Traceroute apprende così l'IP del primo hop.
- TTL=2: il secondo router risponde allo stesso modo.
- E così via finché la destinazione non viene raggiunta, che risponde con ICMP **Port unreachable** (la porta UDP di destinazione è scelta appositamente alta per non trovare servizi in ascolto).

Per ogni TTL vengono inviati **3 probe** (tre colonne di tempi nell'output). La flag `-z 1` imposta il tempo minimo tra una sonda e l'altra in millisecondi.

Per il comando in generale, vedi [traceroute](06_Comandi_Linux.md#traceroute).

---

## Cosa osservare

### 1. Verificare la configurazione degli indirizzi IP

Su ciascuno dei device (`pc1`, `pc2`, `pc3`, `r1`, `r2`):
```bash
ip address
```

Verificare che le interfacce `eth0` (e `eth1` sui router) abbiano l'indirizzo corretto e che il MAC corrisponda a quanto dichiarato in `lab.conf`.

### 2. Verificare il default gateway sugli host

Su `pc1`, `pc2`, `pc3`:
```bash
routel
```

Output atteso su `pc1`:
```
Dst              Gateway        Prefsrc        Protocol  Scope  Dev    Table
default          195.11.14.1                             eth0
195.11.14.0/24                  195.11.14.5    kernel    link   eth0
...
```

La riga `default` con gateway `195.11.14.1` conferma che il default gateway è configurato correttamente.

### 3. Verificare le routing table dei router

Su `r1`:
```bash
routel
```

Output atteso:
```
Dst              Gateway        Prefsrc        Protocol  Scope  Dev    Table
100.0.0.8/30                    100.0.0.9      kernel    link   eth1
195.11.14.0/24                  195.11.14.1    kernel    link   eth0
200.1.1.0/24     100.0.0.10                              eth1
...
```

Le prime due righe (protocol `kernel`, scope `link`) sono le rotte **directly connected**, aggiunte automaticamente all'assegnazione degli indirizzi. La terza è la rotta statica verso LAN C aggiunta manualmente nel `.startup`. Vedi [visualizzare la routing table con routel](07b_Router_Linux.md#visualizzare-la-routing-table-con-routel).

### 4. Ping tra host sulla stessa LAN (pc3 → pc2) e ARP

Connettere Wireshark al collision domain C e avviare la cattura su `eth1`:
```bash
kathara lconfig -n wireshark --add C
# poi su localhost:3000 → selezionare eth1
```

Su `pc3`, verificare la ARP cache prima e dopo il ping:
```bash
arp -n                  # cache inizialmente vuota
ping -c 2 200.1.1.7     # ping verso pc2
arp -n                  # ora contiene il MAC di pc2
```

Output ARP dopo il ping:
```
Address     HWtype  HWaddress           Flags Mask  Iface
200.1.1.7   ether   00:00:00:00:00:02   C           eth0
```

Verificare su `pc2` che anche la sua cache si sia aggiornata (comportamento standard RFC 826: il ricevitore della ARP request apprende il MAC del mittente):
```bash
# su pc2
arp -n
# → 200.1.1.3  ether  00:00:00:00:00:03  C  eth0
```

In Wireshark si osserva la sequenza: ARP request (broadcast), ARP reply, ICMP echo request, ICMP echo reply. Il traffico non attraversa r2: la freccia resta confinata alla LAN C.

### 5. Ping cross-router (pc2 → pc1) e ARP

Connettere Wireshark anche al collision domain B e avviare la cattura su `eth2`:
```bash
kathara lconfig -n wireshark --add B
# su localhost:3000 → selezionare eth2
```

Su `pc2`:
```bash
ping -c 2 195.11.14.5
```

Output atteso:
```
64 bytes from 195.11.14.5: icmp_seq=1 ttl=62 time=5.86 ms
64 bytes from 195.11.14.5: icmp_seq=2 ttl=62 time=1.69 ms
```

> Il TTL nella risposta è 62 (non 64): ogni router decrementa il TTL di 1 durante il forwarding. Il pacchetto attraversa r2 e r1, quindi 64 − 2 = 62.

Verificare la ARP cache di `pc2`:
```bash
arp -n
```

Output atteso:
```
Address     HWtype  HWaddress           Flags Mask  Iface
200.1.1.1   ether   00:00:00:00:00:c1   C           eth0
200.1.1.3   ether   00:00:00:00:00:03   C           eth0
```

La prima riga è il MAC dell'`eth0` di r2: pc2 non può risolvere il MAC di pc1 (LAN diversa), quindi l'ARP va al default gateway. Vedi [comportamento ARP tra reti diverse](07b_Router_Linux.md#comportamento-arp-tra-reti-diverse).

Verificare la ARP cache dei router:
```bash
# su r1
arp -n
# → 195.11.14.5  ether  00:00:00:00:00:01  C  eth0   (pc1)
# → 100.0.0.10   ether  00:00:00:00:00:b2  C  eth1   (eth1 di r2)

# su r2
arp -n
# → 100.0.0.9   ether  00:00:00:00:00:b1  C  eth1   (eth1 di r1)
# → 200.1.1.7   ether  00:00:00:00:00:02  C  eth0   (pc2)
```

In Wireshark su B si vedono le ARP tra i due router e i pacchetti ICMP con IP sorgente/destinazione invariati ma MAC che cambiano a ogni hop.

### 6. Traceroute (pc2 → pc1)

La cattura Wireshark su C (eth1) deve essere attiva. Su `pc2`:
```bash
traceroute 195.11.14.5 -z 1
```

Output atteso:
```
traceroute to 195.11.14.5 (195.11.14.5), 30 hops max, 60 byte packets
 1  200.1.1.1 (200.1.1.1)    0.882 ms  0.662 ms  0.456 ms   ← eth0 di r2
 2  100.0.0.9 (100.0.0.9)    0.903 ms  0.877 ms  1.218 ms   ← eth1 di r1
 3  195.11.14.5 (195.11.14.5) 0.987 ms  1.354 ms  1.015 ms  ← pc1
```

In Wireshark si osservano per ogni TTL: 3 probe UDP seguiti dal rispettivo ICMP Time To Live exceeded. Quando il TTL è sufficiente a raggiungere la destinazione, pc1 risponde con ICMP Port unreachable (la porta UDP è volutamente alta, nessun servizio in ascolto).

---

## Cosa imparare

- Assegnare indirizzi IPv4 e netmask con `ip address add` e configurare il default gateway con `ip route add default`.
- La routing table di un router contiene rotte **directly connected** (aggiunte automaticamente all'assegnazione degli indirizzi) e rotte **statiche** (aggiunte esplicitamente per le reti non direttamente connesse).
- Il traffico tra host sulla stessa LAN non attraversa mai il router: l'ARP viene risolta tra i due host direttamente.
- Quando la destinazione è in una rete diversa, l'host esegue ARP per il MAC del **default gateway**, non della destinazione finale.
- Anche i router mantengono ARP cache per ciascuna interfaccia Ethernet.
- `traceroute` usa probe UDP con TTL crescente; ogni router sul percorso risponde con ICMP Time To Live exceeded. La destinazione finale risponde con ICMP Port unreachable.
- Il TTL nella risposta ICMP di un ping cross-router è inferiore a 64: ogni hop decrementa il TTL di 1.