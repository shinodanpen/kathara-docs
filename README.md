# premessa: concetti base

### Device

Undevice in Katharà è un container Docker che emula un dispositivo di rete (PC, router, switch, ecc.). Può avere un numero arbitrario di interfacce di rete virtuali (`eth0`, `eth1`, ...).

### Collision Domain (dominio di collisione)

Un collision domain è un segmento di rete virtuale condiviso — l'equivalente virtuale di un cavo o di uno switch a cui colleghi più dispositivi.

In Katharà, ogni collision domain è identificato da un nome (es. `A`, `LAN1`, `net_core`). Tutti i device che hanno un'interfaccia collegata allo stesso collision domain si trovano sulla stessa rete virtuale e possono comunicare tra loro a livello L2, a patto che le interfacce siano configurate con indirizzi IP compatibili.

```
kathara vstart -n pc1 --eth 0:A
kathara vstart -n pc2 --eth 0:A
# pc1 e pc2 sono ora sulla stessa rete virtuale "A"
# e possono comunicare (dopo aver configurato gli IP)
```

> **Nota:** Il nome "collision domain" viene dalla terminologia Ethernet classica, dove i dispositivi collegati allo stesso mezzo fisico condividevano il canale e potevano generare collisioni. In Katharà è semplicemente l'etichetta che identifica una rete virtuale.

# parte 1: comandi e parametri

## Kathara – Riferimento Comandi (da man pages ufficiali)

Kathara è un emulatore di reti basato su container Docker. Ogni dispositivo di rete è un container; ogni collegamento è una rete virtuale chiamata **collision domain**. È il successore spirituale di Netkit, con cui è cross-compatibile.

I comandi si dividono in tre famiglie:

- **v-comandi** (`vstart`, `vclean`, `vconfig`): gestione di singoli device
- **l-comandi** (`lstart`, `lclean`, `linfo`, `lrestart`, `lconfig`): gestione di scenari di rete completi
- **comandi globali** (`connect`, `exec`, `wipe`, `list`, `settings`, `check`)

---

## Sintassi generale

```bash
kathara [-h] [-v] <comando> [argomenti]
```

---

## v-comandi (singolo device)

### `kathara vstart` — Avviare un device

```bash
kathara vstart -n DEVICE_NAME [opzioni]
```

Avvia un nuovo device con il nome specificato. I nomi devono essere unici per utente (utenti diversi possono usare lo stesso nome). Senza opzioni, usa i default di `kathara.conf`.

**Opzioni principali:**

|Opzione|Descrizione|
|---|---|
|`-n NOME`|Nome del device (obbligatorio)|
|`--eth N:CD[/MAC]`|Aggiunge un'interfaccia eth_N_ collegata al collision domain _CD_. MAC opzionale nel formato `XX:XX:XX:XX:XX:XX`. N deve essere sequenziale a partire da 0.|
|`--mem MEM`|Limita la RAM (es. `128m`, `1g`). Minimo 4m.|
|`--cpus N`|Limita CPU (float; es. `1.5` su host con 2 CPU).|
|`-i IMAGE`|Usa un'immagine Docker specifica (locale o da Hub).|
|`-e COMANDO`|Esegue un comando dentro il device durante lo startup.|
|`--bridged`|Aggiunge un'interfaccia collegata alla rete host via NAT (configurata automaticamente via DHCP). Sarà l'ultima interfaccia.|
|`--port [HOST:]GUEST[/PROTO]`|Mappa una porta host a una porta del device. Default HOST: 3000, default PROTO: tcp.|
|`--hosthome` / `--no-hosthome`|Monta (o non monta) la home dell'utente dentro il device in `/hosthome`.|
|`--privileged`|Avvia il device in modalità privilegiata (richiede root).|
|`--noterminals` / `--terminals`|Controlla se aprire o meno finestre terminale.|
|`--num_terms N`|Numero di terminali da aprire.|
|`--shell SHELL`|Shell da usare dentro il device (es. `bash`, `sh`).|
|`--xterm APP`|Emulatore di terminale alternativo (solo Unix).|
|`--sysctl OPT`|Imposta opzioni sysctl (solo namespace `net.`).|
|`--env NOME=VALORE`|Imposta variabili d'ambiente.|
|`--ulimit KEY=SOFT[:HARD]`|Imposta ulimit. Usa `-1` per illimitato.|
|`--print`|Dry run: verifica i parametri senza avviare nulla.|

**Esempi:**

```bash
# Device semplice
kathara vstart -n pc1

# Device con RAM limitata e immagine custom
kathara vstart -n mypc1 -i my-image --mem 128m

# Due device collegati sullo stesso collision domain "A"
kathara vstart -n producer --eth 0:A
kathara vstart -n consumer --eth 0:A

# Device con eth0 su dominio A e eth1 connessa alla rete host via NAT
kathara vstart -n router --bridged --eth 0:A

# Dry run
kathara vstart -n test --eth 0:A 1:B --print
```

---

### `kathara vclean` — Fermare un device

```bash
kathara vclean -n DEVICE_NAME
```

Spegne in modo pulito il device specificato.

**Esempio:**

```bash
kathara vclean -n pc1
```

---

### `kathara vconfig` — Gestire interfacce di un device in esecuzione

```bash
kathara vconfig -n DEVICE_NAME (--add <CD[/MAC]> ... | --rm CD ...)
```

Aggiunge o rimuove interfacce di rete da un device già avviato. Il numero della nuova interfaccia viene assegnato automaticamente incrementando l'ultimo usato.

**Opzioni:**

|Opzione|Descrizione|
|---|---|
|`--add CD[/MAC]`|Collega il device al collision domain CD (MAC opzionale).|
|`--rm CD`|Scollega il device dal collision domain CD e rimuove l'interfaccia.|

**Esempi:**

```bash
# Aggiunge interfacce ai domini X e Y (MAC casuali)
kathara vconfig -n pc1 --add X Y

# Aggiunge interfaccia al dominio X con MAC specifico
kathara vconfig -n pc1 --add X/00:00:00:00:00:01

# Rimuove l'interfaccia collegata al dominio X
kathara vconfig -n pc1 --rm X
```

---

## l-comandi (scenari di rete)

Uno **scenario di rete** è una cartella con file di configurazione (`lab.conf`, `lab.dep`, `lab.ext`) e sottocartelle per ogni device. Tutti gli l-comandi operano su questa struttura.

---

### `kathara lstart` — Avviare uno scenario

```bash
kathara lstart [opzioni] [DEVICE_NAME ...]
```

Avvia tutti i device dello scenario. Se si specificano dei nomi, avvia solo quelli. Lo scenario deve essere nella directory corrente (o in quella indicata con `-d`).

**Opzioni principali:**

|Opzione|Descrizione|
|---|---|
|`-d DIRECTORY`|Cartella dello scenario (default: directory corrente).|
|`-F` / `--force-lab`|Forza l'avvio anche senza `lab.conf`.|
|`-l` / `--list`|Mostra info sui device dopo l'avvio.|
|`-o "OPT" ...`|Applica opzioni (stile `lab.conf`) a tutti i device. Es: `--pass "mem=64m" "image=kathara/frr"`.|
|`--exclude NOME ...`|Esclude device specifici dall'avvio.|
|`--hosthome` / `--no-hosthome`|Monta/non monta `/hosthome` nei device.|
|`--shared` / `--no-shared`|Monta/non monta la cartella `shared/` dello scenario in `/shared` nei device.|
|`--noterminals` / `--terminals`|Controlla apertura terminali.|
|`--privileged`|Avvia in modalità privilegiata (richiede root).|
|`--xterm APP`|Emulatore di terminale alternativo.|
|`--print`|Dry run: verifica `lab.conf` senza avviare.|

---

### `kathara lclean` — Fermare uno scenario

```bash
kathara lclean [opzioni] [DEVICE_NAME ...]
```

Spegne tutti i device dello scenario (o solo quelli specificati).

**Opzioni:**

|Opzione|Descrizione|
|---|---|
|`-d DIRECTORY`|Cartella dello scenario.|
|`--exclude NOME ...`|Esclude device specifici dallo spegnimento.|

---

### `kathara lrestart` — Riavviare uno scenario

```bash
kathara lrestart [opzioni] [DEVICE_NAME ...]
```

Esegue `lclean` seguito da `lstart`. Accetta le stesse opzioni di `lstart` (tranne `--print`).

---

### `kathara linfo` — Informazioni sullo scenario

```bash
kathara linfo [opzioni]
```

Mostra lo stato di tutti i device dello scenario corrente. Le colonne visualizzate sono: `LAB_HASH`, `DEVICE_NAME`, `STATUS`, `CPU %`, `MEM USAGE / LIMIT`, `MEM %`, `NET I/O`.

**Opzioni:**

|Opzione|Descrizione|
|---|---|
|`-d DIRECTORY`|Cartella dello scenario.|
|`-w` / `--watch`|Aggiornamento live (uscita con CTRL+C).|
|`-c` / `--conf`|Mostra info statiche da `lab.conf` (numero device, reti, ecc.).|
|`-t` / `--topology`|Mostra la mappatura device ↔ collision domain. Combinabile con `-w`.|
|`-n NOME`|Filtra le info su un solo device.|

---

### `kathara lconfig` — Gestire interfacce in uno scenario

```bash
kathara lconfig [-d DIRECTORY] -n DEVICE_NAME (--add <CD[/MAC]> ... | --rm CD ...)
```

Identico a `vconfig`, ma opera su device appartenenti a uno scenario. Richiede `-n` per specificare il device target.

**Esempi:**

```bash
kathara lconfig -d path/to/scenario -n pc1 --add X Y
kathara lconfig -d path/to/scenario -n pc1 --add X/00:00:00:00:00:01
kathara lconfig -d path/to/scenario -n pc1 --rm X
```

---

## Comandi globali

### `kathara connect` — Aprire una shell su un device

```bash
kathara connect [-d DIRECTORY | -v] [--command SHELL] [-l] DEVICE_NAME
```

Apre una shell interattiva dentro il device specificato.

**Opzioni:**

|Opzione|Descrizione|
|---|---|
|`-d DIRECTORY`|Device appartiene a uno scenario in DIRECTORY.|
|`-v` / `--vdevice`|Device avviato con `vstart`. Non combinabile con `-d`.|
|`--command SHELL`|Comando da eseguire (default: shell da `kathara.conf`).|
|`-l` / `--logs`|Stampa i log di startup prima di aprire la shell.|

**Esempi:**

```bash
# Device avviato con vstart
kathara connect -v pc1

# Device in uno scenario nella directory corrente
kathara connect as1r1
```

---

### `kathara exec` — Eseguire un comando in un device

```bash
kathara exec [-d DIRECTORY | -v] [--no-stdout] [--no-stderr] [--wait] DEVICE_NAME COMANDO
```

Esegue un comando dentro un device senza aprire una shell interattiva.

**Opzioni:**

|Opzione|Descrizione|
|---|---|
|`-d DIRECTORY`|Device in uno scenario in DIRECTORY.|
|`-v` / `--vmachine`|Device avviato con `vstart`.|
|`--no-stdout`|Disabilita stdout del comando.|
|`--no-stderr`|Disabilita stderr del comando.|
|`--wait`|Aspetta che i comandi di startup finiscano (ENTER per forzare).|

**Esempi:**

```bash
# Su device avviato con vstart
kathara exec -v pc1 -- ping 127.0.0.1

# Su device in uno scenario
kathara exec as1r1 "ping 127.0.0.1"
```

---

### `kathara wipe` — Pulizia totale

```bash
kathara wipe [-f] [-s | -a]
```

Spegne **tutti** i device Kathara dell'utente corrente. Con le opzioni si può anche eliminare le impostazioni o agire su tutti gli utenti.

**Opzioni:**

|Opzione|Descrizione|
|---|---|
|`-f` / `--force`|Non chiede conferma prima di procedere.|
|`-s` / `--settings`|Elimina e ricrea `kathara.conf` con valori default. Non compatibile con `-a`.|
|`-a` / `--all`|Pulisce i device di **tutti gli utenti** (richiede root). Non compatibile con `-s`.|

**Esempio:**

```bash
# Resetta solo le impostazioni (non tocca i device)
kathara wipe -s
```

---

### `kathara list` — Elenco device in esecuzione

```bash
kathara list [-a] [-w] [-n DEVICE_NAME]
```

Mostra tutti i device in esecuzione dell'utente corrente con le stesse colonne di `linfo` (`LAB_HASH`, `DEVICE_NAME`, `STATUS`, `CPU %`, `MEM USAGE / LIMIT`, `MEM %`, `NET I/O`).

**Opzioni:**

|Opzione|Descrizione|
|---|---|
|`-a` / `--all`|Mostra device di tutti gli utenti (richiede root).|
|`-w` / `--watch`|Aggiornamento live (CTRL+C per uscire).|
|`-n NOME`|Filtra su un solo device.|

---

### `kathara settings` — Configurazione

```bash
kathara settings
```

Apre un'interfaccia console per visualizzare e modificare tutte le impostazioni in `kathara.conf` (`~/.config/kathara.conf`).

---

### `kathara check` — Verifica ambiente

```bash
kathara check
```

Esegue una serie di test per verificare che l'ambiente sia configurato correttamente. Stampa le versioni di manager, Python, Kathara e OS. Utile anche per allegare info quando si segnala un bug.

**Output atteso:**

```
*  Current Manager is: <your_manager_name>
*  Manager version is: <your_manager_version>
*  Python version is: <python_version_used>
*  Kathara version is: <kathara_version>
*  Operating System version is: <your_os>
*  Trying to run container with <your_default_image> image...
*  Container run successfully.
```

---

## Riepilogo rapido

|Comando|Cosa fa|
|---|---|
|`vstart -n NOME`|Avvia un device singolo|
|`vclean -n NOME`|Ferma un device singolo|
|`vconfig -n NOME --add/--rm`|Aggiunge/rimuove interfacce a device singolo|
|`lstart`|Avvia l'intero scenario|
|`lclean`|Ferma l'intero scenario|
|`lrestart`|Riavvia lo scenario (lclean + lstart)|
|`linfo`|Mostra stato dello scenario|
|`lconfig -n NOME --add/--rm`|Aggiunge/rimuove interfacce in uno scenario|
|`connect NOME`|Apre shell su un device|
|`exec NOME COMANDO`|Esegue comando su un device|
|`wipe`|Spegne tutto (tutti i device dell'utente)|
|`list`|Elenca tutti i device attivi|
|`settings`|Modifica configurazione Kathara|
|`check`|Verifica l'ambiente|

# PARTE 2: STRUTTURA DELLE CARTELLE E DEI LABORATORI

## Network Scenario (Scenario di Rete)

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

---

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
# pc1 parte solo dopo che pc2 e pc3 sono up
pc1: pc2 pc3
# pc3 parte solo dopo pc2
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

---

### `ip link` — Stato interfacce (layer 2)

```
ip link show [DEV]           # mostra stato interfacce (tutte, o solo DEV)
ip link set DEV up           # attiva l'interfaccia DEV
ip link set DEV down         # disattiva l'interfaccia DEV
```

**Esempi:**

```bash
ip link show
ip link show eth0
ip link set eth0 up
ip link set eth0 down
```

---

## Routing

### `ip route` — Gestione tabella di routing

```
ip route [show]                          # mostra la tabella di routing
ip route add NETWORK/MASK via GATEWAY    # aggiunge una route verso una rete
ip route add NETWORK/MASK dev DEV        # aggiunge una route su un'interfaccia diretta
ip route add default via GATEWAY         # aggiunge il default gateway
ip route del NETWORK/MASK [via GATEWAY]  # rimuove una route
```

**Esempi:**

```bash
ip route
ip route add 10.0.0.0/8 via 192.168.1.1
ip route add 10.0.0.0/8 dev eth1
ip route add default via 192.168.1.1
ip route del 10.0.0.0/8 via 192.168.1.1
```

> **Nota:** `ip route add default` e `ip route add 0.0.0.0/0` sono equivalenti.

---

## Test di connettività

### `ping` — Verifica raggiungibilità di un host

```
ping HOST                          # ping continuo verso HOST (CTRL+C per fermare)
ping -c COUNT HOST                 # invia esattamente COUNT pacchetti
ping -i INTERVAL HOST              # imposta l'intervallo tra pacchetti in secondi
ping -W TIMEOUT HOST               # imposta il timeout per ogni risposta in secondi
```

**Esempi:**

```bash
ping 192.168.1.1
ping -c 4 192.168.1.1
ping -c 3 -i 0.5 10.0.0.1
```

---

### `traceroute` — Traccia il percorso verso un host

```
traceroute [-m MAX_HOPS] [-w TIMEOUT] HOST
```

Mostra tutti i router attraversati per raggiungere HOST, con i relativi tempi di risposta.

|Parametro|Descrizione|
|---|---|
|`-m MAX_HOPS`|Numero massimo di hop (default: 30)|
|`-w TIMEOUT`|Timeout in secondi per ogni risposta (default: 5)|

**Esempi:**

```bash
traceroute 10.0.0.2
traceroute -m 10 10.0.0.2
```

> `* * *` su una riga significa che quel router non ha risposto (potrebbe non supportare ICMP o avere il rate limiting attivo).

---

## Analisi del traffico

### `tcpdump` — Cattura e analisi pacchetti

```
tcpdump [-i DEV] [-c COUNT] [-n] [-v] [FILTRO]
```

Cattura e mostra in tempo reale i pacchetti che transitano su un'interfaccia. Senza opzioni ascolta su tutte le interfacce. Interrompi con CTRL+C.

|Parametro|Descrizione|
|---|---|
|`-i DEV`|Interfaccia su cui ascoltare (es. `eth0`, `any` per tutte)|
|`-c COUNT`|Cattura solo COUNT pacchetti poi si ferma|
|`-n`|Non risolve nomi DNS (mostra IP numerici — più veloce e leggibile in lab)|
|`-v` / `-vv`|Output più dettagliato|
|`FILTRO`|Espressione di filtro BPF (vedi esempi)|

**Esempi:**

```bash
tcpdump -i eth0                          # tutto il traffico su eth0
tcpdump -i eth0 -n                       # come sopra, senza risoluzione DNS
tcpdump -i eth0 -n -c 20                 # cattura solo 20 pacchetti
tcpdump -i eth0 host 192.168.1.2         # solo traffico da/verso quell'IP
tcpdump -i eth0 src 192.168.1.2          # solo traffico proveniente da quell'IP
tcpdump -i eth0 dst 10.0.0.1             # solo traffico diretto a quell'IP
tcpdump -i eth0 port 80                  # solo traffico sulla porta 80
tcpdump -i eth0 icmp                     # solo pacchetti ICMP (es. ping)
tcpdump -i eth0 icmp and host 10.0.0.1   # ICMP da/verso un IP specifico
```

> **Filtri utili in lab:** `icmp`, `tcp`, `udp`, `arp`, `host IP`, `port N`, `net NETWORK/MASK` Si possono combinare con `and`, `or`, `not`.

---

## Risoluzione dei nomi

### `cat /etc/hosts` — Mappature statiche hostname → IP

```
cat /etc/hosts                           # mostra le mappature statiche
echo "IP HOSTNAME" >> /etc/hosts         # aggiunge una mappatura
```

**Esempi:**

```bash
cat /etc/hosts
echo "10.0.0.1 router1" >> /etc/hosts
```

---

## Processi e servizi

### `ps` — Lista processi in esecuzione

```
ps aux                                   # mostra tutti i processi con dettagli
ps aux | grep NOME                       # filtra per nome processo
```

---

### `netstat` / `ss` — Connessioni e porte in ascolto

```
ss -tuln                                 # porte TCP/UDP in ascolto (no DNS)
ss -tp                                   # connessioni TCP attive con PID
```

|Flag|Descrizione|
|---|---|
|`-t`|Connessioni TCP|
|`-u`|Connessioni UDP|
|`-l`|Solo porte in ascolto (listening)|
|`-n`|Mostra numeri invece di nomi|
|`-p`|Mostra il processo associato|

---

## Riepilogo rapido

|Comando|Cosa fa|
|---|---|
|`ip addr`|Mostra/gestisce indirizzi IP|
|`ip link set DEV up/down`|Attiva/disattiva un'interfaccia|
|`ip route`|Mostra/gestisce la tabella di routing|
|`ping [-c COUNT] [-i INTERVAL] [-W TIMEOUT] HOST`|Testa la raggiungibilità|
|`traceroute HOST`|Traccia il percorso verso un host|
|`tcpdump -i DEV`|Cattura traffico su un'interfaccia|
|`ss -tuln`|Mostra porte in ascolto|

# PARTE 4: CONDIVISIONE DI FILE TRA HOST E DEVICES

Ci sono due modi per scambiare file tra il filesystem dell'host e quello dei device.

---

### File mirrored (sincronizzati in tempo reale)

Le modifiche su un lato si riflettono automaticamente sull'altro. Ci sono due directory mirror disponibili:

|Directory nel device|Corrisponde a|Default|
|---|---|---|
|`/shared`|cartella `shared/` nella cartella del lab|**abilitata**|
|`/hosthome`|home directory dell'utente sull'host|disabilitata|

`/shared` è la modalità più comoda per scambiare file durante un lab attivo — basta mettere o leggere file da `shared/` nella cartella del lab sull'host.

`/hosthome` va abilitata esplicitamente con `--hosthome` in `lstart`/`vstart` o nelle impostazioni.

---

### File copiati (copie indipendenti)

I file messi nelle sottocartelle dei device nella cartella del lab vengono **copiati** dentro il device all'avvio. Le modifiche successive su un lato non si riflettono sull'altro.

La struttura delle sottocartelle viene mappata sulla root `/` del device:

```
lab/
└── pc1/
    └── prova/
        └── file.txt   →   copiato in /prova/file.txt dentro pc1
```

> Questa modalità è utile per pre-caricare file di configurazione prima dell'avvio, mentre `/shared` è più adatta allo scambio dinamico durante il lab.

# PARTE 5: configurazione e comandi dei dispositivi di rete

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

# PARTE 6: WIRESHARK in katharà

Wireshark è un analizzatore di rete (sniffer) che cattura e ispeziona il traffico su un'interfaccia di rete. In Katharà viene eseguito come device dedicato, accessibile tramite browser dalla macchina host.

---

### Configurazione nel `lab.conf`

Wireshark viene dichiarato come un device speciale nel `lab.conf` usando un'immagine Docker preconfigurata con interfaccia web:

```
wireshark[bridged]=true
wireshark[port]="3000:3000"
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[num_terms]=0
```

|Opzione|Valore|Descrizione|
|---|---|---|
|`bridged`|`true`|Collega il device alla rete host via NAT. Necessario per renderlo raggiungibile dal browser.|
|`port`|`"3000:3000"`|Mappa la porta 3000 del device sulla porta 3000 dell'host.|
|`image`|`"lscr.io/linuxserver/wireshark"`|Immagine Docker con Wireshark e interfaccia web integrata.|
|`num_terms`|`0`|Non apre terminali: il device si usa esclusivamente via browser.|

---

### Accesso all'interfaccia web

Dopo `kathara lstart`, aprire un browser sulla macchina host e navigare su:

```
localhost:3000
```

---

### Collegare Wireshark ai collision domain

**Non è possibile** collegare Wireshark ai collision domain tramite `lab.conf`: l'opzione `bridged=true` occupa `eth0`, che viene usata per l'interfaccia di rete verso l'host. Le interfacce aggiuntive vanno aggiunte manualmente **dopo** l'avvio del lab, con `lconfig`:

```bash
kathara lconfig -n wireshark --add CD
```

```bash
kathara lconfig -n wireshark --add A
kathara lconfig -n wireshark --add B
```

> Le interfacce aggiunte con `lconfig` vengono assegnate a partire da `eth1` (poiché `eth0` è già occupata dall'interfaccia bridged). Il primo collision domain aggiunto sarà su `eth1`, il secondo su `eth2`, e così via.

Una volta connesso, selezionare l'interfaccia corrispondente (`eth1`, `eth2`, ...) nell'interfaccia web di Wireshark per avviare la cattura sul collision domain desiderato.
