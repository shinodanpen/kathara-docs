# Bridge Linux

← [07_Dispositivi_Rete](07_Dispositivi_Rete.md) | [01_INDEX](01_INDEX.md)

Un bridge Linux è uno **switch software** che opera a livello 2 (MAC). Quando un device Katharà viene configurato come bridge, intercetta tutto il traffico Ethernet sulle interfacce collegate e lo instrada in base ai MAC address appresi dinamicamente, esattamente come farebbe uno switch hardware.

L'immagine `kathara/base` è sufficiente. Il comportamento da bridge si ottiene interamente tramite i comandi nel file `.startup`. Vedi [04_Struttura_Laboratorio](04_Struttura_Laboratorio.md#immagini-docker).

---

## Creare e configurare un bridge

### 1. Creare il bridge

```bash
ip link add name BRIDGE_NAME type bridge
```

Crea un nuovo oggetto bridge con il nome interno specificato. Il bridge viene creato in stato DOWN.

```bash
ip link add name mainbridge type bridge
```

> `BRIDGE_NAME` è il nome interno usato dal kernel Linux per riferirsi al bridge. È distinto dal nome del device Katharà (es. `b1`) che identifica il container.

---

### 2. Collegare le interfacce al bridge (enslaving)

```bash
ip link set dev DEV master BRIDGE_NAME
```

Collega l'interfaccia `DEV` al bridge. Va ripetuto per ogni interfaccia.

```bash
ip link set dev eth0 master mainbridge
ip link set dev eth1 master mainbridge
ip link set dev eth2 master mainbridge
ip link set dev eth3 master mainbridge
```

> Le porte del bridge vengono numerate a partire da 1 nell'ordine in cui vengono collegate (non corrispondono necessariamente al numero dell'interfaccia). Un bridge Linux supporta fino a 1024 porte (limite hardcoded nel kernel).

---

### 3. Attivare il bridge

```bash
ip link set up dev BRIDGE_NAME
```

Porta il bridge in stato UP. Senza questo comando il bridge esiste ma non funziona.

---

### 4. Impostare l'ageing time (opzionale)

```bash
brctl setageing BRIDGE_NAME SECONDS
```

Imposta per quanti secondi un MAC address viene mantenuto nel FDB dopo l'ultimo pacchetto ricevuto da quella sorgente. Il default è 300 secondi (5 minuti).

```bash
brctl setageing mainbridge 600
```

---

## File `.startup` — esempio completo

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

## Filtering Database (FDB)

Il FDB è la tabella che il bridge usa per decidere dove inoltrare i frame Ethernet. Per ogni MAC address apprende su quale porta si trova, così da poter fare **forwarding selettivo** invece di broadcast.

### Visualizzare il FDB

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

| Colonna | Descrizione |
|---|---|
| `port no` | Numero della porta virtuale del bridge (assegnato in ordine di enslaving, parte da 1) |
| `mac addr` | Indirizzo MAC |
| `is local?` | `yes` = interfaccia locale del bridge stesso; `no` = MAC appreso dinamicamente da un host esterno |
| `ageing timer` | Secondi dall'ultimo pacchetto con quel MAC sorgente. `0.00` per le entry locali (non scadono mai) |

### Comportamento del learning

Il bridge apprende i MAC address leggendo il **campo sorgente** di ogni frame che arriva su una porta. Impara quindi solo i **mittenti**, non i destinatari.

- Quando riceve un frame verso un MAC **non ancora nel FDB**: lo inoltra in broadcast su tutte le porte tranne quella di arrivo (**flooding**).
- Una volta che il MAC di destinazione è nel FDB: il frame viene inoltrato **solo sulla porta corretta**.
- Le entry dinamiche scadono dopo `ageing time` secondi di inattività. Al pacchetto successivo da quel MAC, il bridge riparte con il flooding.

---

## `scapy` — Generare pacchetti Ethernet raw

`scapy` è uno strumento Python preinstallato in `kathara/base` che permette di costruire e inviare pacchetti di rete a basso livello. Utile per testare il bridge learning senza dover configurare IP.

```bash
scapy        # avvia la shell interattiva
```

**Costruire e inviare un frame Ethernet:**

```python
p = Ether(dst='00:00:00:00:00:02', src='00:00:00:00:00:01')
sendp(p, iface='eth0')
```

> Questo è sufficiente per far apprendere al bridge il MAC sorgente sulla porta corrispondente, anche senza payload IP.

Per catturare il traffico generato e verificare il comportamento del bridge, vedi [tcpdump](06_Comandi_Linux.md#tcpdump) o [08_Wireshark](08_Wireshark.md).

---

## Note sulla configurazione in `lab.conf`

Per disabilitare il traffico NDP (IPv6) che potrebbe interferire con l'osservazione del bridge learning:

```
b1[ipv6]="false"
```

Vedi [lab.conf](04_Struttura_Laboratorio.md#labconf) per le altre opzioni disponibili.
