# Laboratorio 1 — Primo lab Katharà

← [09_Laboratori](09_Laboratori.md) | [01_INDEX](01_INDEX.md)

Primo contatto con Katharà: installazione dell'ambiente, struttura di un lab, configurazione degli indirizzi IP e verifica della connettività tra macchine sullo stesso dominio di collisione.

---

## Topologia
```
           20.1.1.0/24                    20.1.2.0/24
                                                       
  pc1 ──────────[A]──────────── pc2 ────────[B]──────── pc3
 eth0                          eth0  eth1             eth0
20.1.1.2                    20.1.1.1  20.1.2.1      20.1.2.5
```

**Prefissi di rete:**

| Collision domain | Prefisso        |
|---|---|
| A                | 20.1.1.0/24     |
| B                | 20.1.2.0/24     |

**Indirizzi IP:**

| Device | Interfaccia | Collision domain | Indirizzo IP  |
|---|---|---|---|
| pc1    | eth0        | A                | 20.1.1.2/24   |
| pc2    | eth0        | A                | 20.1.1.1/24   |
| pc2    | eth1        | B                | 20.1.2.1/24   |
| pc3    | eth0        | B                | 20.1.2.5/24   |

> **Nota:** pc1 e pc3 non sono direttamente raggiungibili tra loro — si trovano su collision domain diversi e non c'è ancora un router configurato. Vedi [07b_Router_Linux](07b_Router_Linux.md) per il forwarding IP.

---

## File di configurazione

### `lab.conf`
```
pc1[0]="A"      # eth0 di pc1 → collision domain A
pc2[0]="A"      # eth0 di pc2 → collision domain A
pc2[1]="B"      # eth1 di pc2 → collision domain B (pc2 è su entrambe le LAN)
pc3[0]="B"      # eth0 di pc3 → collision domain B
```

### `pc1.startup`
```bash
ip address add 20.1.1.2/24 dev eth0
# /24 significa che i primi 24 bit identificano la rete: tutti i device
# con indirizzo 20.1.1.x/24 sono sulla stessa LAN e si raggiungono direttamente
```

### `pc2.startup`
```bash
ip address add 20.1.1.1/24 dev eth0   # interfaccia verso la LAN A
ip address add 20.1.2.1/24 dev eth1   # interfaccia verso la LAN B
# pc2 ha un piede in entrambe le reti, ma NON fa routing:
# forwarding IP non è abilitato, quindi non instrada pacchetti tra A e B
```

### `pc3.startup`
```bash
ip address add 20.1.2.5/24 dev eth0
```

---

## Prerequisiti di installazione

Prima di avviare qualsiasi lab, l'ambiente deve essere configurato correttamente. L'ordine di installazione è rilevante: Katharà dipende da Docker.

1. **VSCode** — editor consigliato per modificare `lab.conf` e file `.startup`. Download: [code.visualstudio.com](https://code.visualstudio.com/download). Durante l'installazione su Windows è utile abilitare le opzioni *"Apri con Code"* nel menu contestuale e *"Aggiungi a PATH"*.

2. **Docker** — motore container su cui girano i device Katharà. Senza Docker avviato, Katharà non parte.
   - Windows / macOS: Docker Desktop
   - Linux (Debian): seguire la guida ufficiale [docs.docker.com/engine/install](https://docs.docker.com/engine/install/)

3. **Katharà** — download da [kathara.org/download.html](https://www.kathara.org/download.html). Su Windows SmartScreen potrebbe bloccare l'eseguibile: cliccare *"Ulteriori informazioni"* → *"Esegui comunque"*.

**Verifica dell'ambiente:**
```bash
kathara check
```

Se non compaiono errori, Docker e Katharà sono correttamente installati e in esecuzione. Vedi [kathara check](03_Comandi_Kathara.md#check) per l'output atteso.

---

## Concetti chiave

### Notazione degli indirizzi IP nei diagrammi

Nei diagrammi di rete usati nel corso, ogni collision domain ha un **prefisso** associato (es. `20.1.1.0/24`). I numeri nei box accanto alle singole interfacce rappresentano solo l'**ultimo byte** dell'indirizzo IP completo.

Esempio: prefisso `20.1.1.0/24`, ultimo byte `2` → indirizzo completo `20.1.1.2/24`.

Il `/24` indica che i primi 24 bit (i primi 3 byte) identificano la rete. Tutti i device con lo stesso prefisso `/24` si trovano sulla stessa LAN e si raggiungono direttamente a livello L3, senza bisogno di un router. Il significato completo della notazione CIDR sarà approfondito più avanti nel corso.

### Limiti del ping su reti diverse senza routing

`ping` opera a livello 3 (IP). Due macchine su collision domain diversi (es. pc1 su A e pc3 su B) hanno prefissi di rete incompatibili: pc1 sa che `20.1.2.x` non è nella sua rete locale, ma non ha nessuna rotta configurata per raggiungerla. Il pacchetto viene scartato prima ancora di uscire dall'host. Per far comunicare reti diverse serve un router con forwarding IP abilitato — vedi [07b_Router_Linux](07b_Router_Linux.md#abilitare-il-forwarding-ip).

---

## Cosa osservare

### 1. Avvio del lab
```bash
cd primo-lab/
kathara lstart
```

Katharà apre un terminale per ogni device. Verificare che tutti e tre siano attivi.

### 2. Verifica degli indirizzi configurati

Su ciascun device:
```bash
ip addr show
```

Output atteso su pc2 (esempio):
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 20.1.1.1/24 scope global eth0
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 20.1.2.1/24 scope global eth1
```

### 3. Ping tra macchine sullo stesso collision domain

**Da pc1 → pc2 (entrambi su A):**
```bash
ping -c 3 20.1.1.1
```

Output atteso:
```
PING 20.1.1.1 (20.1.1.1) 56(84) bytes of data.
64 bytes from 20.1.1.1: icmp_seq=1 ttl=64 time=0.XXX ms
64 bytes from 20.1.1.1: icmp_seq=2 ttl=64 time=0.XXX ms
64 bytes from 20.1.1.1: icmp_seq=3 ttl=64 time=0.XXX ms
--- 20.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```

**Da pc2 → pc3 (entrambi su B):**
```bash
ping -c 3 20.1.2.5
```

Output atteso: identico alla forma precedente, con `20.1.2.5` come destinazione.

**Da pc1 → pc3 (collision domain diversi, senza routing):**
```bash
ping -c 3 20.1.2.5
```

Output atteso:
```
connect: Network is unreachable
```

oppure nessuna risposta con tutti i pacchetti persi — conferma che senza routing le reti non comunicano.

### 4. Arresto del lab
```bash
kathara lclean
```

---

## Cosa imparare

- Un lab Katharà è una cartella con `lab.conf`, file `.startup` e sottocartelle per i device. Vedi [04_Struttura_Laboratorio](04_Struttura_Laboratorio.md).
- Il `lab.conf` definisce la topologia: quale interfaccia di quale device è collegata a quale collision domain. Vedi [lab.conf](04_Struttura_Laboratorio.md#labconf).
- Gli indirizzi IP si assegnano nei file `.startup` con `ip address add IP/MASK dev DEV`. Vedi [ip addr](06_Comandi_Linux.md#interfacce-di-rete).
- Due device sullo stesso collision domain con indirizzi IP compatibili si raggiungono direttamente con `ping`. Vedi [ping](06_Comandi_Linux.md#ping).
- Due device su collision domain diversi **non** si raggiungono senza un router con forwarding IP abilitato — il prefisso di rete incompatibile rende la destinazione irraggiungibile già al livello del mittente.
- `kathara check` è il comando per verificare che l'ambiente sia funzionante. Vedi [kathara check](03_Comandi_Kathara.md#check).