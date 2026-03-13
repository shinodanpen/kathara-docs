# Laboratorio 4 — Basic IPv6

← [09_Laboratori](09_Laboratori.md) | [01_INDEX](01_INDEX.md)

**Sorgente:** `kathara-lab_basic-ipv6` v2.1 — Computer Networks Research Group, Roma Tre University

---

## Topologia

Due router (r1, r2) e tre host (pc1, pc2, pc3), collegati tramite tre LAN.
```
pc1 ── [A] ── r1 ── [B] ── r2 ── [C] ── pc2
                                   └── [C] ── pc3
```

### Prefissi IPv6

| Collision domain | Prefisso |
|---|---|
| A | `2001:0:0:1::/64` |
| B | `2001:0:0:2::/64` |
| C | `2001:0:0:3::/64` |

Gli indirizzi di pc1, pc2 e pc3 **non** sono configurati manualmente: vengono assegnati via [SLAAC](#slaac-e-router-advertisement).

### MAC address

| Device | Interfaccia | Collision domain | MAC |
|---|---|---|---|
| pc1 | eth0 | A | `00:00:00:00:00:01` |
| pc2 | eth0 | C | `00:00:00:00:00:02` |
| pc3 | eth0 | C | `00:00:00:00:00:03` |
| r1 | eth0 | A | `00:00:00:00:00:a1` |
| r1 | eth1 | B | `00:00:00:00:00:b1` |
| r2 | eth0 | C | `00:00:00:00:00:c1` |
| r2 | eth1 | B | `00:00:00:00:00:b2` |

I MAC sono forzati per rendere deterministici gli indirizzi link-local e le entry nella neighbor cache.

---

## File di configurazione

### `lab.conf`
```conf
r1[0]="A/00:00:00:00:00:a1"
r1[1]="B/00:00:00:00:00:b1"
r1[image]="kathara/base"
r1[ipv6]="True"

r2[0]="C/00:00:00:00:00:c1"
r2[1]="B/00:00:00:00:00:b2"
r2[image]="kathara/base"
r2[ipv6]="True"

pc1[0]="A/00:00:00:00:00:01"
pc1[image]="kathara/base"
pc1[ipv6]="True"
pc1[sysctl]="net.ipv6.conf.eth0.accept_ra=2"

pc2[0]="C/00:00:00:00:00:02"
pc2[image]="kathara/base"
pc2[ipv6]="True"
pc2[sysctl]="net.ipv6.conf.eth0.accept_ra=2"

pc3[0]="C/00:00:00:00:00:03"
pc3[image]="kathara/base"
pc3[ipv6]="True"
pc3[sysctl]="net.ipv6.conf.eth0.accept_ra=2"

wireshark[bridged]=true
wireshark[port]="3000:3000"
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[num_terms]=0
```

`ipv6="True"` abilita IPv6 sul device; vedi [lab.conf](04_Struttura_Laboratorio.md#labconf).

`sysctl="net.ipv6.conf.eth0.accept_ra=2"` forza gli host ad accettare i Router Advertisement anche quando il forwarding IP è attivo — necessario perché SLAAC funzioni correttamente.

I file `.startup` di pc1, pc2 e pc3 sono **vuoti**: indirizzi e default gateway arrivano da SLAAC, non richiedono configurazione manuale.

### `r1.startup`
```bash
ip address add 2001:0:0:1::1/64 dev eth0
ip address add 2001:0:0:2::1/64 dev eth1

ip route add 2001:0:0:3::/64 via fe80::200:ff:fe00:b2 dev eth1

chmod o-rw /etc/radvd.conf
systemctl start radvd
```

Le prime due righe assegnano indirizzi statici e aggiungono implicitamente le rotte directly connected; vedi [07b_Router_Linux](07b_Router_Linux.md#configurazione-delle-interfacce).

La rotta verso la LAN C usa come nexthop l'**indirizzo link-local** di r2 su B (`fe80::200:ff:fe00:b2`). In IPv6 i nexthop su link Ethernet devono essere link-local, e per questo il comando richiede anche `dev eth1` — un indirizzo link-local non è univoco globalmente, quindi il kernel ha bisogno di sapere su quale interfaccia cercarlo.

`chmod o-rw` e `systemctl start radvd` preparano e avviano il [demone radvd](#slaac-e-router-advertisement).

### `r1/etc/radvd.conf`

Questo file viene copiato in `/etc/radvd.conf` dentro r1 all'avvio (vedi [file copiati](05_Condivisione_File.md#file-copiati-copie-indipendenti)):
```
interface eth0
{
    AdvSendAdvert on;
    MinRtrAdvInterval 3;
    MaxRtrAdvInterval 9;
    AdvDefaultLifetime 27;
    prefix 2001:0:0:1::/64 {};
};
```

| Direttiva | Significato |
|---|---|
| `interface eth0` | Interfaccia su cui inviare i Router Advertisement |
| `AdvSendAdvert on` | Abilita l'invio periodico |
| `MinRtrAdvInterval 3` | Secondi minimi tra RA consecutivi |
| `MaxRtrAdvInterval 9` | Secondi massimi tra RA consecutivi |
| `AdvDefaultLifetime 27` | Per quanti secondi gli host considerano r1 un default gateway valido |
| `prefix 2001:0:0:1::/64 {}` | Prefisso annunciato; gli host lo usano per costruire l'indirizzo globale via SLAAC |

### `r2.startup` e `r2/etc/radvd.conf`

Configurazione simmetrica a r1:
```bash
# r2.startup
ip address add 2001:0:0:3::1/64 dev eth0
ip address add 2001:0:0:2::2/64 dev eth1

ip route add 2001:0:0:1::/64 via fe80::200:ff:fe00:b1 dev eth1

chmod o-rw /etc/radvd.conf
systemctl start radvd
```
```
# r2/etc/radvd.conf
interface eth0
{
    AdvSendAdvert on;
    MinRtrAdvInterval 3;
    MaxRtrAdvInterval 9;
    AdvDefaultLifetime 27;
    prefix 2001:0:0:3::/64 {};
};
```

---

## Concetti chiave

### SLAAC e Router Advertisement

**SLAAC** (Stateless Address Auto-Configuration) permette a un host di configurarsi senza DHCP:

1. Il router invia periodicamente un **Router Advertisement** (RA) ICMPv6 con il prefisso della rete
2. L'host combina il prefisso ricevuto con un **interface ID** derivato dal proprio MAC address tramite il procedimento EUI-64: `00:00:00:00:00:01` → `0200:00ff:fe00:0001` → indirizzo completo `2001:0:0:1::200:ff:fe00:1/64`
3. Dal RA l'host impara anche il default gateway: è l'indirizzo link-local del router mittente

`radvd` è il demone Linux che invia i messaggi RA. Gira solo sui router. La sua configurazione (`radvd.conf`) specifica su quale interfaccia annunciare e quale prefisso includere.

### Indirizzi link-local

Ogni interfaccia IPv6 si assegna automaticamente un indirizzo nel prefisso `fe80::/10`, detto **link-local**. Questo indirizzo è valido solo sulla rete locale, non viene ruotato tra reti diverse, ed è usato per NDP e come nexthop nelle rotte statiche.

L'indirizzo link-local si costruisce anch'esso con EUI-64: `MAC 00:00:00:00:00:b2` → `fe80::200:ff:fe00:b2`.

### NDP e Neighbor Cache

In IPv6 non esiste ARP. Il **Neighbor Discovery Protocol** (NDP) svolge la stessa funzione usando messaggi ICMPv6:

- **Neighbor Solicitation (NS)** — "chi ha questo indirizzo IPv6?" (equivalente ARP request). Viene inviato verso un indirizzo **multicast solicited-node** invece che in broadcast, riducendo il traffico.
- **Neighbor Advertisement (NA)** — risponde con il MAC address (equivalente ARP reply)

Il risultato è memorizzato nella **neighbor cache**, visibile con:
```bash
ip neigh
```

Gli stati principali sono `REACHABLE` (raggiungibilità confermata di recente), `STALE` (entry presente, non verificata di recente), `DELAY` (in attesa di conferma).

Analogia con ARP: la neighbor cache sta a IPv6 come la ARP cache sta a IPv4. Vedi [ARP nei router](07b_Router_Linux.md#arp-nei-router) per il confronto.

---

## Cosa osservare

### Indirizzi IPv6 assegnati

Su r1 e r2, verificare gli indirizzi statici e gli indirizzi link-local automatici:
```bash
ip -6 address
```

Su pc1, pc2, pc3, verificare l'indirizzo SLAAC (scope `global dynamic`) e link-local (scope `link`):
```bash
ip -6 address
```

### Routing table IPv6
```bash
routel -6
```

Sui router: le rotte directly connected (protocol `kernel`) e quella statica verso la LAN non connessa, con nexthop link-local. Vedi [routel](07b_Router_Linux.md#visualizzare-la-routing-table-con-routel).

Sugli host: la rotta `default` appresa via RA, con protocol `ra` e nexthop pari all'indirizzo link-local del router.

### Neighbor cache — ping tra host sulla stessa LAN (pc3 → pc2)

Su pc3, ispezionare la neighbor cache prima del ping:
```bash
ip neigh
```

Eseguire il ping (l'indirizzo di pc2 è calcolato da EUI-64 sul prefisso `2001:0:0:3::/64`):
```bash
ping 2001::3:200:ff:fe00:2
```

Dopo il ping, ispezionare di nuovo `ip neigh` su pc3 e su pc2. La comunicazione è bidirezionale: pc2 apprende il MAC di pc3 dal campo sorgente del NS ricevuto, senza inviarne uno proprio.

### Wireshark — LAN C
```bash
kathara lconfig -n wireshark --add C
```

Aprire `localhost:3000` e catturare su `eth1`. La sequenza attesa è: Router Advertisement periodici → NS da pc3 verso il multicast solicited-node di pc2 → NA da pc2 → Echo request/reply. Il traffico rimane interamente su C: nessun pacchetto attraversa i router. Vedi [08_Wireshark](08_Wireshark.md).

### Neighbor cache — ping tra LAN diverse (pc2 → pc1)
```bash
kathara lconfig -n wireshark --add B
```

Eseguire da pc2:
```bash
ping 2001::1:200:ff:fe00:1
```

Su pc2, verificare con `ip neigh` che la cache contenga il MAC di **r2** (non di pc1): il pacchetto esce dalla LAN C verso r2, che lo inoltra verso r1 e poi a pc1. Su Wireshark catturare su `eth2` (LAN B) per vedere gli NS/NA tra r1 e r2 e gli echo request/reply che attraversano il link.

### traceroute
```bash
# su pc2
traceroute 2001::1:200:ff:fe00:1 -z 1
```

L'output mostra tre hop: eth0 di r2 (`2001:0:0:3::1`), eth1 di r1 (`2001:0:0:2::1`), poi pc1. L'opzione `-z 1` imposta un intervallo minimo di 1 ms tra i probe. Su Wireshark (LAN C) si vedono pacchetti UDP con Hop Limit decrescente e le corrispondenti risposte ICMPv6 **Time Exceeded**.

---

## Cosa imparare

- In IPv6 gli indirizzi possono essere configurati manualmente o via SLAAC; SLAAC usa i Router Advertisement del router e il MAC address dell'host
- I nexthop nelle rotte IPv6 su link Ethernet sono sempre indirizzi link-local; il comando `ip route add` richiede quindi anche `dev` per disambiguare l'interfaccia
- NDP sostituisce ARP con messaggi ICMPv6 NS/NA, usando multicast solicited-node invece di broadcast
- La neighbor cache IPv6 si ispeziona con `ip neigh` (non con `arp -n`)
- Il traffico tra host sulla stessa LAN non attraversa mai il router, indipendentemente dal fatto che gli indirizzi siano globali
- `traceroute` in IPv6 usa UDP con Hop Limit incrementale; i router rispondono con ICMPv6 Time Exceeded