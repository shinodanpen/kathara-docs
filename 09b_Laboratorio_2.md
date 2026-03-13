# Laboratorio 2 — One Bridge

← [09_Laboratori](09_Laboratori.md) | [01_INDEX](01_INDEX.md)

Un [bridge](07a_Bridge_Linux.md) Linux con quattro porte collega quattro PC, ciascuno su un [collision domain](02_Premessa.md#collision-domain) separato. Nessun indirizzo IP viene configurato: si lavora esclusivamente a livello 2, inviando frame Ethernet raw con `scapy`. L'obiettivo è osservare il bridge learning passo passo — controllare esattamente quando il bridge fa flooding e quando smette di farlo.

---

## Topologia
```
pc1 [MAC :01]                     pc4 [MAC :04]
       |                                 |
      (A)                               (D)
       |                                 |
     eth0                             eth3
  [MAC:b1]                         [MAC:b4]
       +----------[  b1  ]----------+
                  |        |
               eth1        eth2
            [MAC:b2]    [MAC:b3]
                  |        |
                 (B)      (C)
                  |        |
           pc2 [MAC :02]  pc3 [MAC :03]
```

| Device | Interfaccia | Collision Domain | MAC address |
|---|---|:---:|---|
| pc1 | eth0 | A | `00:00:00:00:00:01` |
| pc2 | eth0 | B | `00:00:00:00:00:02` |
| pc3 | eth0 | C | `00:00:00:00:00:03` |
| pc4 | eth0 | D | `00:00:00:00:00:04` |
| b1  | eth0 | A | `00:00:00:00:00:b1` |
| b1  | eth1 | B | `00:00:00:00:00:b2` |
| b1  | eth2 | C | `00:00:00:00:00:b3` |
| b1  | eth3 | D | `00:00:00:00:00:b4` |

Mapping porte bridge (dipende dall'ordine di enslaving in `b1.startup`):

| Porta bridge | Interfaccia b1 | Collision Domain | PC raggiunto |
|:---:|---|:---:|---|
| 1 | eth0 | A | pc1 |
| 2 | eth1 | B | pc2 |
| 3 | eth2 | C | pc3 |
| 4 | eth3 | D | pc4 |

> **Nota:** la colonna `port no` nel [FDB](07a_Bridge_Linux.md#filtering-database-fdb) riflette questo mapping, non il numero dell'interfaccia `ethX`.

---

## File di configurazione

### `lab.conf`
```
# ── PC ──────────────────────────────────────────────────────────────────────
pc1[0]="A/00:00:00:00:00:01"    # MAC fisso: rende il FDB leggibile senza ambiguità
pc1[image]="kathara/base"
pc1[ipv6]="false"               # disabilitare IPv6 evita il NDP spontaneo che
                                 # farebbe apprendere il bridge prima del nostro intervento

pc2[0]="B/00:00:00:00:00:02"
pc2[image]="kathara/base"
pc2[ipv6]="false"

pc3[0]="C/00:00:00:00:00:03"
pc3[image]="kathara/base"
pc3[ipv6]="false"

pc4[0]="D/00:00:00:00:00:04"
pc4[image]="kathara/base"
pc4[ipv6]="false"

# ── Bridge ───────────────────────────────────────────────────────────────────
b1[0]="A/00:00:00:00:00:b1"    # l'ordine delle righe determina la numerazione delle porte:
b1[1]="B/00:00:00:00:00:b2"    # b1[0] → porta 1, b1[1] → porta 2, ecc.
b1[2]="C/00:00:00:00:00:b3"
b1[3]="D/00:00:00:00:00:b4"
b1[image]="kathara/base"
b1[ipv6]="false"

# ── Wireshark ────────────────────────────────────────────────────────────────
wireshark[bridged]=true
wireshark[port]="3000:3000"
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[num_terms]=0          # nessun terminale: si accede solo via browser
```

> I PC non hanno file `.startup`: senza IP da configurare e senza servizi da avviare, non serve nulla. Il comportamento di ogni macchina è determinato esclusivamente dai frame inviati manualmente con `scapy`.

### `b1.startup`
```bash
ip link add name mainbridge type bridge   # crea l'oggetto bridge (stato DOWN)

# enslaving: l'ordine conta — eth0 diventa porta 1, eth1 porta 2, ecc.
ip link set dev eth0 master mainbridge
ip link set dev eth1 master mainbridge
ip link set dev eth2 master mainbridge
ip link set dev eth3 master mainbridge

ip link set up dev mainbridge             # senza questo il bridge esiste ma non funziona

brctl setageing mainbridge 600            # 600s invece dei 300s di default —
                                           # dà più tempo per osservare il timer prima della scadenza
```

---

## Concetti chiave

### IPv6 e NDP: perché disabilitarlo

Con IPv6 abilitato, ogni interfaccia si auto-configura un indirizzo link-local e inizia immediatamente a scambiare messaggi **NDP** (Neighbor Discovery Protocol) con le altre macchine sulla stessa rete. Questi messaggi sono frame Ethernet ordinari: il bridge li riceve, legge il MAC sorgente e aggiorna il [FDB](07a_Bridge_Linux.md#filtering-database-fdb) in automatico. Il risultato è che il bridge ha già imparato tutti i MAC address ancora prima che l'utente esegua il primo comando, rendendo impossibile osservare il learning da zero. Disabilitare IPv6 con `[ipv6]="false"` congela la situazione: il FDB inizia vuoto (solo i MAC locali del bridge) e si aggiorna solo quando si invia traffico intenzionalmente. Vedi [lab.conf → ipv6](04_Struttura_Laboratorio.md#labconf).

### Porta bridge vs numero interfaccia

Nel [FDB](07a_Bridge_Linux.md#filtering-database-fdb) la colonna `port no` indica la **porta virtuale del bridge**, non il numero dell'interfaccia di rete. Le porte vengono numerate da 1 nell'ordine in cui i comandi `ip link set dev ... master mainbridge` vengono eseguiti. Cambiare l'ordine di enslaving cambia il mapping porta–dominio, anche se i nomi delle interfacce rimangono gli stessi. In questo lab: porta 1 = eth0 = dominio A, porta 2 = eth1 = dominio B, e così via.

### Il bridge apprende solo i MAC sorgente

Il bridge legge il **campo sorgente** di ogni frame in arrivo — mai quello di destinazione. La conseguenza diretta: quando pc1 invia a pc2, il bridge impara che pc1 è raggiungibile dalla porta su cui è arrivato il frame, ma non impara nulla su pc2. pc2 entra nel FDB solo quando è pc2 stesso a trasmettere. Questa asimmetria è il motivo per cui il flooding continua finché entrambi i lati di una conversazione non hanno parlato almeno una volta. Vedi [07a_Bridge_Linux → Comportamento del learning](07a_Bridge_Linux.md#comportamento-del-learning).

---

## Cosa osservare

### Setup
```bash
# Avviare il lab
kathara lstart

# Collegare Wireshark ai collision domain che si vuole monitorare
# (va fatto dopo lstart: eth0 del container wireshark è occupata dall'interfaccia bridged)
kathara lconfig -n wireshark --add B   # → eth1 nel container wireshark
kathara lconfig -n wireshark --add C   # → eth2 nel container wireshark

# Aprire l'interfaccia web: localhost:3000
# Selezionare eth1 per osservare il dominio B, eth2 per il dominio C
```

### FDB all'avvio — solo MAC locali

Sul device `b1`, prima di inviare qualsiasi traffico:
```bash
brctl showmacs mainbridge
```

Output atteso:
```
port no  mac addr           is local?  ageing timer
  1      00:00:00:00:00:b1  yes        0.00
  1      00:00:00:00:00:b1  yes        0.00
  2      00:00:00:00:00:b2  yes        0.00
  2      00:00:00:00:00:b2  yes        0.00
  3      00:00:00:00:00:b3  yes        0.00
  3      00:00:00:00:00:b3  yes        0.00
  4      00:00:00:00:00:b4  yes        0.00
  4      00:00:00:00:00:b4  yes        0.00
```

Solo le interfacce locali del bridge: `is local? = yes`, `ageing timer = 0.00` (non scadono mai). Nessun MAC esterno ancora appreso. Se avessi qui entry `is local? = no`, significherebbe che IPv6 non è stato davvero disabilitato.

### Scenario di learning passo passo

Tutto il traffico viene generato con [scapy](07a_Bridge_Linux.md#scapy--generare-pacchetti-ethernet-raw). Su ciascun PC, avviare scapy e usare la sintassi:
```python
p = Ether(dst='MAC_DESTINAZIONE', src='MAC_SORGENTE')
sendp(p, iface='eth0')
```

---

**Evento 1 — pc1 → pc2, Wireshark su B**
```python
# su pc1
p = Ether(dst='00:00:00:00:00:02', src='00:00:00:00:00:01')
sendp(p, iface='eth0')
```

Wireshark su B (eth1): **cattura il frame** — il bridge non conosce ancora il MAC di pc2, quindi esegue flooding su tutte le porte eccetto quella di ingresso (porta 1). Il frame arriva su B, C e D.

FDB su b1 dopo l'evento:
```
port no  mac addr           is local?  ageing timer
  1      00:00:00:00:00:01  no         18.54   ← pc1 appena appreso, timer in salita
  1      00:00:00:00:00:b1  yes        0.00
  ...
```

> **Perché non c'è una riga per pc2?** pc2 non ha ancora inviato nulla. Il bridge apprende solo i mittenti, non i destinatari.

---

**Evento 2 — pc1 → pc2 ancora, Wireshark su C**

Stessa trasmissione, stessa osservazione su C (eth2 in Wireshark).

Wireshark su C: **cattura il frame** — pc2 è ancora sconosciuto, flooding continua.

---

**Evento 3 — pc2 → pc1 (risposta), Wireshark su C**
```python
# su pc2
p = Ether(dst='00:00:00:00:00:01', src='00:00:00:00:00:02')
sendp(p, iface='eth0')
```

Wireshark su C: **non cattura nulla** — il bridge riceve il frame su porta 2 (dominio B), impara che pc2 è sulla porta 2, e sa già che pc1 è sulla porta 1. Forwarding diretto porta 2 → porta 1. Il frame non passa né per C né per D.

FDB su b1: ora contiene sia `00:00:00:00:00:01` (porta 1) che `00:00:00:00:00:02` (porta 2).

---

**Evento 4 — pc1 → pc2 di nuovo, Wireshark su C**

Wireshark su C: **non cattura nulla** — sia mittente che destinatario sono nel FDB. Forwarding selettivo porta 1 → porta 2. Il traffico tra pc1 e pc2 è ora invisibile a tutti gli altri collision domain.

---

**Evento 5 — pc1 → pc4, Wireshark su C**
```python
# su pc1
p = Ether(dst='00:00:00:00:00:04', src='00:00:00:00:00:01')
sendp(p, iface='eth0')
```

Wireshark su C: **cattura il frame** — pc4 non è ancora nel FDB, flooding su porte 2, 3, 4. Il dominio C (porta 3) viene raggiunto.

---

**Evento 6 — dopo scadenza ageing, pc4 → pc2, Wireshark su C**

Aspettare che le entry dinamiche scadano (ageing time = 600s nel `b1.startup` di questo lab; per accelerare il test si può ridurre con `brctl setageing mainbridge 30`). Verificare su b1 che il FDB sia tornato a contenere solo i MAC locali, poi:
```python
# su pc4
p = Ether(dst='00:00:00:00:00:02', src='00:00:00:00:00:04')
sendp(p, iface='eth0')
```

Wireshark su C: **cattura il frame** — le entry di pc1, pc2, ecc. sono scadute, il bridge non sa più dove si trovano e torna al flooding.

---

## Cosa imparare

- Il bridge apprende i MAC address **leggendo il campo sorgente** dei frame in transito. Un host appare nel FDB solo quando trasmette, non quando riceve.
- Finché il MAC di destinazione non è nel FDB, il frame viene inoltrato in **flooding** su tutte le porte eccetto quella di arrivo. Questo vale anche quando il mittente è già noto.
- Una volta che entrambi i lati di una comunicazione sono appresi, il traffico tra loro diventa **invisibile** agli altri collision domain: nessun flooding, forwarding diretto.
- L'**ageing timer** conta i secondi dall'ultimo frame ricevuto da quel MAC sorgente. Quando raggiunge il valore di ageing time configurato, la entry scompare e il bridge ricomincia a fare flooding per quell'host — esattamente come se non l'avesse mai visto.
- IPv6 va disabilitato ogni volta che si vuole osservare il bridge learning in modo controllato: il traffico NDP automatico lo precluderebbe.
- Le **porte virtuali** del bridge non corrispondono necessariamente ai numeri delle interfacce `ethX`: dipende dall'ordine in cui viene fatto l'enslaving nel file `.startup`.