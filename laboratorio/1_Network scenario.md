Uno scenario di rete è una **cartella** che contiene la descrizione completa di una rete virtuale: quali device esistono, come sono collegati, come si configurano all'avvio. Si lancia con `lstart` da dentro quella cartella (o passando il percorso con `-d`).

---

### Struttura della cartella

```
mio-laboratorio/
│
├── lab.conf              ← topologia e opzioni dei device (quasi sempre obbligatorio)
├── lab.dep               ← dipendenze di avvio tra device (opzionale)
├── lab.ext               ← collegamento a reti fisiche esterne via VLAN (opzionale, solo Linux root)
│
├── pc1/                  ← cartella del device "pc1" (opzionale)
│   └── etc/
│       └── resolv.conf   ← esempio: file copiato in /etc/resolv.conf dentro il device
│
├── pc1.startup           ← script eseguito all'avvio di pc1 (opzionale)
├── pc1.shutdown          ← script eseguito allo spegnimento di pc1 (opzionale)
│
├── shared/               ← cartella condivisa tra host e tutti i device, montata in /shared (opzionale)
├── shared.startup        ← script eseguito su TUTTI i device prima del loro .startup (opzionale)
└── shared.shutdown       ← script eseguito su TUTTI i device dopo il loro .shutdown (opzionale)
```

> **Regola dei file nelle sottocartelle device:** la cartella `pc1/` viene trattata come se fosse la root `/` del device. Un file in `pc1/etc/resolv.conf` finirà in `/etc/resolv.conf` dentro il container.

---

### Quali file servono, quando

|File/Cartella|Obbligatorio?|Quando usarlo|
|---|---|---|
|`lab.conf`|Quasi sempre sì|Definisce la topologia. Senza, `lstart` non parte (a meno di `-F`).|
|`lab.dep`|No|Solo se certi device devono aspettare che altri siano già up prima di avviarsi.|
|`lab.ext`|No|Solo se vuoi connettere un collision domain a una rete fisica reale (Linux + root).|
|`device/`|No|Solo se devi pre-popolare il filesystem del device con dei file.|
|`device.startup`|No|Quasi sempre utile: configura IP, avvia demoni, ecc.|
|`device.shutdown`|No|Raramente necessario.|
|`shared/`|No|Utile per scambiare file tra host e device durante il lab.|
|`shared.startup`|No|Se hai operazioni comuni da fare su tutti i device all'avvio.|
|`shared.shutdown`|No|Se hai operazioni comuni da fare su tutti i device allo spegnimento.|

---

### `lab.conf` — il file più importante

Sintassi: `device[opzione]=valore`

---

#### Collegare interfacce a collision domain

Il numero tra parentesi quadre indica l'interfaccia (`eth0`, `eth1`, ...). Il valore è il nome del collision domain, con MAC opzionale.

```
pc1[0]="A"                        # eth0 di pc1 → collision domain A
pc1[1]="B"                        # eth1 di pc1 → collision domain B
r1[0]="A"                         # eth0 di r1  → collision domain A
r1[1]="C/02:42:ac:11:00:02"       # eth1 di r1  → dominio C con MAC specifico
```

---

#### Opzioni per device

Ogni opzione si scrive come `device[opzione]=valore` e si applica solo al device specificato.

|Opzione|Valore|Descrizione|
|---|---|---|
|`[image]`|`"nome:tag"`|Immagine Docker da usare come sistema base. Default: `kathara/base`.|
|`[ipv6]`|`"true"` / `"false"`|Abilita o disabilita IPv6. Default: `"true"`. Disabilitarlo evita il traffico NDP automatico che verrebbe generato all'avvio e interferirebbe con scenari in cui si vuole osservare il learning del bridge passo dopo passo.|
|`[mem]`|`"128m"`, `"1g"`, ...|Limita la RAM del container. Minimo `4m`.|
|`[cpus]`|`0.5`, `1.5`, ...|Limita l'uso di CPU (valore float rispetto ai core disponibili sull'host).|
|`[shell]`|`"/bin/bash"`|Shell usata da Katharà per aprire i terminali e per eseguire i file `.startup`.|
|`[exec]`|`"comando"`|Comando aggiuntivo eseguito dentro il device durante lo startup, dopo il file `.startup`.|
|`[bridged]`|`true`|Aggiunge un'interfaccia collegata alla rete host via NAT, configurata automaticamente via DHCP. Utile per dare accesso a internet al device o per esporre servizi verso l'host.|
|`[port]`|`"HOST:GUEST/PROTO"`|Mappa una porta dell'host a una porta del device. `PROTO` può essere `tcp` o `udp` (default: `tcp`). Es: `"8080:80/tcp"`, `"3000:3000"`.|
|`[num_terms]`|`0`, `1`, `2`, ...|Numero di finestre terminale da aprire all'avvio del device. `0` non apre nessun terminale.|
|`[sysctl]`|`"net.X.Y=valore"`|Imposta un parametro sysctl nel namespace `net.`. Tipicamente usato per abilitare il forwarding IP su router.|
|`[privileged]`|`"true"`|Avvia il container in modalità privilegiata (accesso completo alle risorse dell'host). Richiede che Katharà sia avviato come root.|
|`[hosthome]`|`"true"` / `"false"`|Monta la home dell'utente host in `/hosthome` dentro il device.|
|`[shared]`|`"true"` / `"false"`|Monta la cartella `shared/` dello scenario in `/shared` dentro il device.|
|`[env]`|`"NOME=VALORE"`|Imposta una variabile d'ambiente nel container.|
|`[ulimit]`|`"KEY=SOFT[:HARD]"`|Imposta un ulimit. Usa `-1` per illimitato.|
#### Immagini Docker

L'immagine standard è `kathara/base`: una macchina Linux minimale con strumenti di rete preinstallati (`iproute2`, `tcpdump`, `ping`, `scapy`, ecc.). Viene usata indifferentemente per PC, router e bridge — il comportamento del device dipende dai comandi nel file `.startup`, non dall'immagine.

Per scenari particolari esistono immagini specializzate:

|Immagine|Uso|
|---|---|
|`kathara/base`|Immagine standard. PC, router, bridge.|
|`kathara/frr`|Routing dinamico con FRRouting (OSPF, BGP, ecc.).|
|`lscr.io/linuxserver/wireshark`|Istanza di Wireshark accessibile via browser.|

---

#### Esempi

```
# PC base con MAC fisso e IPv6 disabilitato
pc1[0]="A/00:00:00:00:00:01"
pc1[image]="kathara/base"
pc1[ipv6]="false"

# Router con forwarding IPv4 abilitato e RAM limitata
r1[0]="A"
r1[1]="B"
r1[image]="kathara/base"
r1[mem]="256m"
r1[sysctl]="net.ipv4.ip_forward=1"

# Wireshark accessibile via browser su localhost:3000, senza terminale
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[bridged]=true
wireshark[port]="3000:3000"
wireshark[num_terms]=0
```

---

#### Metadati dello scenario

Opzionali, vengono mostrati all'avvio del lab.

```
LAB_NAME="Esempio"
LAB_DESCRIPTION="Rete semplice con un router e due PC"
LAB_VERSION=1.0
LAB_AUTHOR="Mario"
```

---

### `lab.dep` — dipendenze di avvio

Garantisce che certi device vengano avviati solo dopo altri. Utile quando un device deve trovare già attivo un router o un server.

```
pc1 parte solo dopo che pc2 e pc3 sono up
pc1: pc2 pc3
pc3 parte solo dopo pc2
pc3: pc2
```

---

### `lab.ext` — reti esterne (avanzato)

Collega un collision domain a un'interfaccia fisica dell'host, con supporto VLAN. Solo Linux, solo root. Da usare con cautela.

```
# Dominio A sull'interfaccia enp9s0 (no VLAN)
A enp9s0
# Dominio B su enp9s0 con VLAN ID 20
B enp9s0.20
# Dominio C su eth1 con VLAN ID 4001
C eth1.4001
```

> Quando `lab.ext` è presente, i terminali non si aprono automaticamente. Usare `kathara connect` per accedere ai device.

---

### Esempio completo: rete con un router e due PC

**Topologia:** `pc1 — [A] — r1 — [B] — pc2`

```
mio-lab/
├── lab.conf
├── r1.startup
├── pc1.startup
└── pc2.startup
```

`lab.conf`:

```
r1[0]="A"
r1[1]="B"
pc1[0]="A"
pc2[0]="B"
```

`r1.startup`:

```bash
ip addr add 192.168.1.1/24 dev eth0
ip addr add 10.0.0.1/24 dev eth1
```

`pc1.startup`:

```bash
ip addr add 192.168.1.2/24 dev eth0
ip route add default via 192.168.1.1
```

`pc2.startup`:

````bash
ip addr add 10.0.0.2/24 dev eth0
ip route add default via 10.0.0.1
```# PARTE 3: comandi e configurazione base dei devices linux

I device Katharà sono container Linux con strumenti di rete preinstallati (`iproute2`).

> **Nota:** i comandi `ifconfig` e `route` (vecchio stile) potrebbero non essere disponibili. Usa sempre `ip` al loro posto.

---

## Interfacce di rete

### `ip addr` — Gestione indirizzi IP


ip addr [show [DEV]]         # mostra indirizzi (tutti, o solo DEV)
ip addr add IP/MASK dev DEV  # assegna un indirizzo alla interfaccia DEV
ip addr del IP/MASK dev DEV  # rimuove un indirizzo dalla interfaccia DEV
````

**Esempi:**

```bash
ip addr
ip addr show eth0
ip addr add 192.168.1.1/24 dev eth0
ip addr del 192.168.1.1/24 dev eth0
```
