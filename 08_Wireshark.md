# Wireshark in Katharà

← [01_INDEX](01_INDEX.md)

Wireshark è un analizzatore di rete (sniffer) che cattura e ispeziona il traffico su un'interfaccia di rete. In Katharà viene eseguito come **device dedicato**, accessibile tramite browser dalla macchina host.

---

## Configurazione nel `lab.conf`

Wireshark va dichiarato come un device speciale nel [lab.conf](04_Struttura_Laboratorio.md#labconf) usando un'immagine Docker preconfigurata con interfaccia web:

```
wireshark[bridged]=true
wireshark[port]="3000:3000"
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[num_terms]=0
```

| Opzione | Valore | Descrizione |
|---|---|---|
| `bridged` | `true` | Collega il device alla rete host via NAT. Necessario per renderlo raggiungibile dal browser. |
| `port` | `"3000:3000"` | Mappa la porta 3000 del device sulla porta 3000 dell'host. |
| `image` | `"lscr.io/linuxserver/wireshark"` | Immagine Docker con Wireshark e interfaccia web integrata. |
| `num_terms` | `0` | Non apre terminali: il device si usa esclusivamente via browser. |

---

## Accesso all'interfaccia web

Dopo `kathara lstart`, aprire un browser sulla macchina host e navigare su:

```
localhost:3000
```

---

## Collegare Wireshark ai collision domain

**Non è possibile** collegare Wireshark ai [collision domain](02_Premessa.md#collision-domain) tramite `lab.conf`: l'opzione `bridged=true` occupa `eth0`, usata per l'interfaccia verso l'host. Le interfacce aggiuntive vanno aggiunte manualmente **dopo** l'avvio del lab, con [lconfig](03_Comandi_Kathara.md#lconfig):

```bash
kathara lconfig -n wireshark --add CD
```

```bash
kathara lconfig -n wireshark --add A
kathara lconfig -n wireshark --add B
```

> Le interfacce aggiunte con `lconfig` vengono assegnate a partire da `eth1` (poiché `eth0` è già occupata dall'interfaccia bridged). Il primo collision domain aggiunto sarà su `eth1`, il secondo su `eth2`, e così via.

Una volta connessa, selezionare l'interfaccia corrispondente (`eth1`, `eth2`, ...) nell'interfaccia web per avviare la cattura sul collision domain desiderato.

---

## Lettura del traffico

> *Sezione da completare — da integrare con i laboratori specifici che usano Wireshark.*

Argomenti da trattare:
- Navigare l'interfaccia web
- Filtri di cattura e di visualizzazione
- Lettura di frame Ethernet, pacchetti IP, segmenti TCP/UDP
- Identificare ARP, ICMP, e altri protocolli rilevanti nei lab

Per la cattura testuale da riga di comando vedi [tcpdump](06_Comandi_Linux.md#tcpdump).
