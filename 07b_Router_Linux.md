# Router Linux

← [[07_Dispositivi_Rete]] | [[01_INDEX]]

Un router Linux opera a **livello 3** (IP). A differenza del [[07a_Bridge_Linux|bridge]] che lavora sui MAC address a livello 2, il router prende decisioni di forwarding basandosi sugli indirizzi IP e su una tabella di routing.

L'immagine `kathara/base` è sufficiente. Il comportamento da router si ottiene interamente tramite i comandi nel file `.startup`. Vedi [[04_Struttura_Laboratorio#immagini-docker]].

---

## Configurazione delle interfacce

Ogni interfaccia del router deve ricevere un indirizzo IP con la relativa netmask. Per i comandi di base vedi anche [[06_Comandi_Linux#interfacce-di-rete|ip addr]].

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

Un device Linux di default **non** inoltra pacchetti tra interfacce diverse. Per farlo funzionare come router, il forwarding IP va abilitato esplicitamente. Il modo più pulito è tramite [[04_Struttura_Laboratorio#labconf|lab.conf]]:

```
r1[sysctl]="net.ipv4.ip_forward=1"
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

Per i comandi generali sulla routing table vedi [[06_Comandi_Linux#routing|ip route]].

---

## Visualizzare la routing table con `routel`

```bash
routel
```

`routel` è un frontend leggibile per `ip route`. Mostra la routing table in formato tabulare con le colonne `Dst`, `Gateway`, `Prefsrc`, `Protocol`, `Scope`, `Dev`, `Table`.

**Esempio di output su un router:**

```
Dst              Gateway      Prefsrc       Protocol  Scope  Dev
10.0.0.0/30                   10.0.0.1      kernel    link   eth1
192.168.1.0/24                192.168.1.1   kernel    link   eth0
200.1.1.0/24     10.0.0.2                             eth1
10.0.0.1         10.0.0.1     kernel        host       eth1   local
192.168.1.1      192.168.1.1  kernel        host       eth0   local
```

- Le rotte con `Protocol kernel` e scope `link` sono le rotte **directly connected**, aggiunte automaticamente all'assegnazione dell'indirizzo.
- Le rotte senza `Protocol` e senza `Prefsrc` sono quelle aggiunte manualmente con `ip route add`.

---

## ARP nei router

Anche i router eseguono ARP ogni volta che devono inviare pacchetti su una rete Ethernet. Hanno quindi la propria ARP cache, ispezionabile con:

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

### Comportamento ARP tra reti diverse

Quando un host vuole raggiungere un indirizzo IP **esterno** alla propria rete locale, inoltra il pacchetto al default gateway. L'ARP request viene quindi emessa per risolvere il MAC del **router** (non quello del destinatario finale, irraggiungibile a livello 2). La ARP cache dell'host conterrà il MAC dell'interfaccia del router, non quello dell'host di destinazione.

I router a loro volta eseguono ARP su ciascuna interfaccia per risolvere il MAC del next-hop verso cui inoltrano i pacchetti.

> Il traffico tra host sulla **stessa rete locale** non attraversa mai il router: l'ARP viene risolta direttamente tra i due host, e il MAC di destinazione è quello dell'host stesso.

---

## File `.startup` — esempio completo

```bash
ip address add 192.168.1.1/24 dev eth0
ip address add 10.0.0.1/30 dev eth1
ip route add 200.1.1.0/24 via 10.0.0.2 dev eth1
```

> Le prime due righe assegnano indirizzi IP alle interfacce (aggiungendo implicitamente le rotte directly connected). La terza aggiunge una rotta statica verso una rete non direttamente connessa, specificando come next-hop l'indirizzo del router adiacente sull'altro lato del link.

---

## Note sulla configurazione in `lab.conf`

Esempio di router con forwarding IP e RAM limitata:

```
r1[0]="A"
r1[1]="B"
r1[image]="kathara/base"
r1[mem]="256m"
r1[sysctl]="net.ipv4.ip_forward=1"
```

Vedi [[04_Struttura_Laboratorio#labconf|lab.conf]] per le altre opzioni disponibili.
