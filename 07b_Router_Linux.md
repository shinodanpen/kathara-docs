# Router Linux

← [07_Dispositivi_Rete](07_Dispositivi_Rete.md) | [01_INDEX](01_INDEX.md)

Un router Linux opera a **livello 3** (IP). A differenza del [bridge](07a_Bridge_Linux.md) che lavora sui MAC address a livello 2, il router prende decisioni di forwarding basandosi sugli indirizzi IP e su una tabella di routing.

L'immagine `kathara/base` è sufficiente. Il comportamento da router si ottiene interamente tramite i comandi nel file `.startup`. Vedi [04_Struttura_Laboratorio](04_Struttura_Laboratorio.md#immagini-docker).

---

## Configurazione delle interfacce

Ogni interfaccia del router deve ricevere un indirizzo IP con la relativa netmask. Per i comandi di base vedi anche [ip addr](06_Comandi_Linux.md#interfacce-di-rete).

```bash
ip address add IP/MASK dev DEV
```

```bash
ip address add 192.168.1.1/24 dev eth0
ip address add 10.0.0.1/30 dev eth1
```

> Quando si assegna un indirizzo a un'interfaccia, il kernel aggiunge automaticamente una rotta verso la rete direttamente connessa (**directly connected**, scope `link`). Non è necessario aggiungere manualmente le rotte per le reti direttamente connesse.

---

## Abilitare il forwarding IP

Un device Linux di default **non** inoltra pacchetti tra interfacce diverse. Per farlo funzionare come router, il forwarding IP va abilitato esplicitamente. Il modo più pulito è tramite [lab.conf](04_Struttura_Laboratorio.md#labconf):

```
r1[sysctl]="net.ipv4.ip_forward=1"
r1[sysctl]="net.ipv6.conf.all.forwarding=1"
```

Oppure direttamente nel `.startup` o nella shell del device:

```bash
sysctl -w net.ipv4.ip_forward=1
```

---

## Rotte statiche verso reti non direttamente connesse

Per raggiungere reti che non sono direttamente connesse al router occorre aggiungere una rotta statica, indicando il **next-hop** (gateway) attraverso cui inoltrarla.

```bash
ip route add NETWORK/MASK via GATEWAY dev DEV
```

```bash
ip route add 200.1.1.0/24 via 10.0.0.2 dev eth1
```

Per il **default gateway** degli host (traffico verso qualsiasi destinazione non coperta da rotte più specifiche):

```bash
ip route add default via GATEWAY
ip route add default via 192.168.1.1
```

In IPv6, il next-hop è tipicamente un **indirizzo link-local** (prefisso `fe80::`), e in quel caso l'interfaccia di uscita (`dev DEV`) è obbligatoria, perché gli indirizzi link-local non sono globalmente unici.

```bash
ip route add 2001:0:0:3::/64 via fe80::200:ff:fe00:b2 dev eth1
```

Per i comandi generali sulla routing table vedi [ip route](06_Comandi_Linux.md#routing).

---

## Visualizzare la routing table con `routel`

```bash
routel # routing table IPv4 routel -6 # routing table IPv6
```

`routel` è un frontend leggibile per `ip route`. Con `-6` mostra solo le rotte IPv6, incluse le rotte directly connected, le link-local, i prefissi multicast (`ff00::/8`) e le rotte statiche con next-hop link-local.

**Esempio di output su un router:**

```
Dst              Gateway      Prefsrc       Protocol  Scope  Dev
10.0.0.0/30                   10.0.0.1      kernel    link   eth1
192.168.1.0/24                192.168.1.1   kernel    link   eth0
200.1.1.0/24     10.0.0.2                             eth1
10.0.0.1         10.0.0.1     kernel        host       eth1   local
192.168.1.1      192.168.1.1  kernel        host       eth0   local
```

> Le rotte con gateway `fe80::...` sono rotte statiche IPv6 con next-hop link-local. Il campo `Dev` è obbligatorio in questi casi.

- Le rotte con `Protocol kernel` e scope `link` sono le rotte **directly connected**, aggiunte automaticamente all'assegnazione dell'indirizzo.
- Le rotte senza `Protocol` e senza `Prefsrc` sono quelle aggiunte manualmente con `ip route add`.

---

## ARP e Neighbor Discovery nei router

### ARP (IPv4)

I router eseguono ARP ogni volta che devono inviare pacchetti IPv4 su una rete Ethernet. La ARP cache si ispeziona con:
```bash
arp -n        # mostra la ARP cache senza risolvere i nomi via DNS
```

| Colonna | Descrizione |
|---|---|
| `Address` | Indirizzo IP |
| `HWtype` | Tipo di hardware (di solito `ether`) |
| `HWaddress` | MAC address corrispondente |
| `Flags` | `C` = entry appresa dinamicamente |
| `Iface` | Interfaccia su cui è stata risolta |

---

### Neighbor Discovery (IPv6)

In IPv6, ARP è sostituito dal **Neighbor Discovery Protocol (NDP)**, basato su ICMPv6. La tabella equivalente si chiama **neighbor cache** e si ispeziona con:
```bash
ip neigh        # mostra la neighbor cache (IPv4 e IPv6)
```

**Esempio di output:**
```
fe80::200:ff:fe00:b2 dev eth1 lladdr 00:00:00:00:00:b2 router STALE
fe80::200:ff:fe00:1  dev eth0 lladdr 00:00:00:00:00:01 router STALE
2001::1:200:ff:fe00:1 dev eth0 lladdr 00:00:00:00:00:01 router STALE
```

| Campo | Descrizione |
|---|---|
| Indirizzo IPv6 | Indirizzo del vicino (globale o link-local) |
| `dev` | Interfaccia su cui è stato risolto |
| `lladdr` | MAC address corrispondente |
| `router` | Il vicino è un router |
| Stato | `REACHABLE` → raggiungibile; `STALE` → entry valida ma non verificata di recente; `DELAY` → in attesa di verifica |

NDP usa messaggi **Neighbor Solicitation (NS)** e **Neighbor Advertisement (NA)** (visibili in Wireshark come ICMPv6) esattamente come ARP usa request e reply.

### Comportamento tra reti diverse (IPv4 e IPv6)

Quando un host vuole raggiungere un indirizzo **esterno** alla propria rete locale, il pacchetto viene inoltrato al default gateway. La risoluzione dell'indirizzo fisico (ARP o NDP) avviene quindi per il **router**, non per la destinazione finale.

> Il traffico tra host sulla **stessa rete locale** non attraversa mai il router: la risoluzione avviene direttamente tra i due host.

---

## radvd — Router Advertisement Daemon

`radvd` è un demone che invia periodicamente messaggi **Router Advertisement (RA)** su un'interfaccia. Gli host che li ricevono possono usarli per configurarsi automaticamente tramite **SLAAC** (Stateless Address Auto-Configuration): derivano il proprio indirizzo IPv6 globale combinando il prefisso annunciato con il proprio MAC address (formato EUI-64), e imparano il default gateway.

### Configurazione in `lab.conf`

Per far accettare i Router Advertisement a un host (necessario in Katharà perché il forwarding IPv6 è attivo):
```
pc1[sysctl]="net.ipv6.conf.eth0.accept_ra=2"
```

> Il valore `2` forza l'accettazione degli RA anche se il forwarding IPv6 è abilitato sul device.

### File `radvd.conf`

Il file di configurazione sta in `/etc/radvd.conf` dentro il container del router. Va pre-popolato nella cartella del device prima dell'avvio (vedi [05_Condivisione_File](05_Condivisione_File.md)).
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

| Parametro | Descrizione |
|---|---|
| `interface` | Interfaccia del router su cui inviare gli RA |
| `AdvSendAdvert on` | Abilita l'invio periodico di RA |
| `MinRtrAdvInterval` | Intervallo minimo tra RA consecutivi (secondi) |
| `MaxRtrAdvInterval` | Intervallo massimo tra RA consecutivi (secondi) |
| `AdvDefaultLifetime` | Durata di validità del default gateway comunicata agli host (secondi) |
| `prefix` | Prefisso IPv6 annunciato per SLAAC |

### Avvio nel `.startup`
```bash
chmod o-rw /etc/radvd.conf    # permessi richiesti da radvd
systemctl start radvd
```

---

## File `.startup` — esempio completo

**IPv4:**
```bash
ip address add 192.168.1.1/24 dev eth0
ip address add 10.0.0.1/30 dev eth1
ip route add 200.1.1.0/24 via 10.0.0.2 dev eth1
```

**IPv6 (con radvd):**
```bash
ip address add 2001:0:0:1::1/64 dev eth0
ip address add 2001:0:0:2::1/64 dev eth1
ip route add 2001:0:0:3::/64 via fe80::200:ff:fe00:b2 dev eth1
chmod o-rw /etc/radvd.conf
systemctl start radvd
```

> In IPv6 il next-hop della rotta statica è l'indirizzo link-local del router adiacente su quella LAN. La specifica di `dev` è obbligatoria per le rotte con next-hop link-local.


---

## Note sulla configurazione in `lab.conf`

Esempio di router con forwarding IP e RAM limitata:

```
r1[0]="A"
r1[1]="B"
r1[image]="kathara/base"
r1[mem]="256m"
r1[sysctl]="net.ipv4.ip_forward=1"
r1[sysctl]="net.ipv6.conf.all.forwarding=1"
```

Per un router IPv6 con radvd e forwarding abilitato:
```
r1[0]="A/00:00:00:00:00:a1"
r1[1]="B/00:00:00:00:00:b1"
r1[image]="kathara/base"
r1[ipv6]="True"
r1[sysctl]="net.ipv6.conf.all.forwarding=1"
```

> Il MAC address fisso (`/XX:XX:XX:XX:XX:XX`) è utile con IPv6 perché il MAC viene incorporato nell'indirizzo link-local tramite EUI-64 — con MAC prevedibili, gli indirizzi link-local diventano leggibili e costanti tra riavvii.

Vedi [lab.conf](04_Struttura_Laboratorio.md#labconf) per le altre opzioni disponibili.
