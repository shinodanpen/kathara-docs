## 5.1 Bridge Linux

Un bridge Linux è uno switch software che opera a livello 2 (MAC). Quando un device Katharà viene configurato come bridge, intercetta tutto il traffico Ethernet sulle interfacce collegate e lo instrada in base ai MAC address appresi dinamicamente, esattamente come farebbe uno switch hardware.

L'immagine `kathara/base` è sufficiente: non serve un'immagine dedicata. Il comportamento da bridge si ottiene interamente tramite i comandi nel file `.startup`.

---

### Creare e configurare un bridge

**1. Creare il bridge:**

```bash
ip link add name BRIDGE_NAME type bridge
```

Crea un nuovo oggetto bridge con il nome interno specificato. Il bridge viene creato in stato DOWN.

```bash
ip link add name mainbridge type bridge
```

> `BRIDGE_NAME` è il nome interno usato dal software Linux per riferirsi al bridge. È distinto dal nome del device Katharà (es. `b1`) che identifica il container.

---

**2. Collegare le interfacce al bridge (enslaving):**

```bash
ip link set dev DEV master BRIDGE_NAME
```

Collega l'interfaccia `DEV` al bridge. L'operazione va ripetuta per ogni interfaccia.

```bash
ip link set dev eth0 master mainbridge
ip link set dev eth1 master mainbridge
ip link set dev eth2 master mainbridge
ip link set dev eth3 master mainbridge
```

> Le porte del bridge vengono numerate a partire da 1 nell'ordine in cui vengono collegate (non corrispondono necessariamente al numero dell'interfaccia). Un bridge Linux supporta fino a 1024 porte (limite hardcoded nel kernel).

---

**3. Attivare il bridge:**

```bash
ip link set up dev BRIDGE_NAME
```

Porta il bridge in stato UP. Senza questo comando il bridge esiste ma non funziona.

---

**4. Impostare l'ageing time (opzionale):**

```bash
brctl setageing BRIDGE_NAME SECONDS
```

Imposta per quanti secondi un MAC address viene mantenuto nel FDB dopo l'ultimo pacchetto ricevuto da quella sorgente. Il default è 300 secondi (5 minuti).

```bash
brctl setageing mainbridge 600
```

---

### File `b1.startup` — esempio completo

```bash
ip link add name mainbridge type bridge
ip link set dev eth0 master mainbridge
ip link set dev eth1 master mainbridge
ip link set dev eth2 master mainbridge
ip link set dev eth3 master mainbridge
ip link set up dev mainbridge
brctl setageing mainbridge 600
```

---

### Filtering Database (FDB)

Il FDB è la tabella che il bridge usa per decidere dove inoltrare i frame Ethernet. Per ogni MAC address apprende su quale porta si trova, così da poter fare forwarding selettivo invece di broadcast.

**Visualizzare il FDB:**

```bash
brctl showmacs BRIDGE_NAME
```

**Esempio di output:**

```
port no  mac addr           is local?  ageing timer
  1      00:00:00:00:00:b1  yes        0.00
  2      00:00:00:00:00:b2  yes        0.00
  1      00:00:00:00:00:01  no         18.54
```

**Colonne:**

|Colonna|Descrizione|
|---|---|
|`port no`|Numero della porta virtuale del bridge (assegnato in ordine di enslaving, parte da 1)|
|`mac addr`|Indirizzo MAC|
|`is local?`|`yes` = interfaccia locale del bridge stesso; `no` = MAC appreso dinamicamente da un host esterno|
|`ageing timer`|Secondi dall'ultimo pacchetto con quel MAC sorgente. `0.00` per le entry locali (non scadono mai); incrementa per le entry dinamiche|

**Comportamento del learning:**

Il bridge apprende i MAC address leggendo il campo sorgente di ogni frame che arriva su una porta. Impara quindi solo i mittenti, non i destinatari. Quando riceve un frame verso un MAC non ancora nel FDB, lo inoltra in broadcast su tutte le porte tranne quella di arrivo (flooding). Una volta che il MAC di destinazione è nel FDB, il frame viene inoltrato solo sulla porta corretta.

Le entry dinamiche scadono dopo `ageing time` secondi di inattività. Al pacchetto successivo da quel MAC, il bridge riparte con il flooding.

---

### `scapy` — Generare pacchetti Ethernet raw

`scapy` è uno strumento Python preinstallato in `kathara/base` che permette di costruire e inviare pacchetti di rete a basso livello, utile per testare il comportamento del bridge senza dover configurare IP.

```bash
scapy        # avvia la shell interattiva di scapy
```

**Costruire e inviare un frame Ethernet:**

```python
p = Ether(dst='00:00:00:00:00:02', src='00:00:00:00:00:01')
sendp(p, iface='eth0')
```

> Questo è sufficiente per far apprendere al bridge il MAC sorgente sulla porta corrispondente, anche senza payload.

## 5.2 Router Linux

Un router Linux opera a livello 3 (IP). A differenza del bridge che lavora sui MAC address a livello 2, il router prende decisioni di forwarding basandosi sugli indirizzi IP e su una tabella di routing.

L'immagine `kathara/base` è sufficiente: non serve un'immagine dedicata. Il comportamento da router si ottiene interamente tramite i comandi nel file `.startup`.

---

### Configurazione delle interfacce

Ogni interfaccia del router deve ricevere un indirizzo IP con la relativa netmask. Assegnando un indirizzo a un'interfaccia, il kernel aggiunge automaticamente una rotta verso la rete direttamente connessa.

```bash
ip address add IP/MASK dev DEV
```

```bash
ip address add 192.168.1.1/24 dev eth0
ip address add 10.0.0.1/30 dev eth1
```

> Quando si assegna un indirizzo a un'interfaccia, la rete corrispondente viene aggiunta automaticamente alla routing table come rotta **directly connected** (scope `link`). Non è necessario aggiungere manualmente le rotte per le reti direttamente connesse.

---

### Rotte statiche verso reti non direttamente connesse

Per raggiungere reti che non sono direttamente connesse al router occorre aggiungere una rotta statica, indicando il next-hop (gateway) attraverso cui inoltrarla.

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

---

### Visualizzare la routing table con `routel`

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

Le rotte con `Protocol kernel` e scope `link` sono le rotte directly connected, aggiunte automaticamente all'assegnazione dell'indirizzo. Le rotte senza `Protocol` e senza `Prefsrc` sono quelle aggiunte manualmente con `ip route add`.

---

### ARP nei router

Anche i router eseguono ARP ogni volta che devono inviare pacchetti su una rete Ethernet. Hanno quindi la propria ARP cache, ispezionabile con:

```bash
arp -n                     # mostra la ARP cache senza risolvere i nomi via DNS
```

**Colonne dell'output:**

|Colonna|Descrizione|
|---|---|
|`Address`|Indirizzo IP|
|`HWtype`|Tipo di hardware (di solito `ether`)|
|`HWaddress`|MAC address corrispondente|
|`Flags`|`C` = entry appresa dinamicamente|
|`Iface`|Interfaccia su cui è stata risolta|

**Comportamento ARP tra reti diverse:**

Quando un host vuole raggiungere un indirizzo IP esterno alla propria rete locale, inoltra il pacchetto al default gateway. L'ARP request viene quindi emessa per risolvere il MAC del router (non quello del destinatario finale, irraggiungibile a livello 2). La ARP cache dell'host conterrà perciò il MAC dell'interfaccia del router, non quello dell'host di destinazione.

I router a loro volta eseguono ARP su ciascuna interfaccia per risolvere il MAC del next-hop verso cui inoltrano i pacchetti.

> Il traffico tra host sulla stessa rete locale non attraversa mai il router: l'ARP viene risolta direttamente tra i due host, e il MAC di destinazione è quello dell'host stesso.

---

### File `.startup` — esempio completo

```bash
ip address add 192.168.1.1/24 dev eth0
ip address add 10.0.0.1/30 dev eth1
ip route add 200.1.1.0/24 via 10.0.0.2 dev eth1
```

> Le prime due righe assegnano indirizzi IP alle interfacce (aggiungendo implicitamente le rotte directly connected). La terza aggiunge una rotta statica verso una rete non direttamente connessa, specificando come next-hop l'indirizzo del router adiacente sull'altro lato del link.