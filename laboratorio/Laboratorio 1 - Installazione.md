
> **Corso:** Reti di Calcolatori  
> **Fonte:** Slide laboratorio – Lezione 1  
> **Tags:** #kathara #laboratorio #installazione #reti

---
## Installazione degli strumenti

### VSCode

- Scaricare da: https://code.visualstudio.com/download
- Scegliere la versione adatta al proprio sistema operativo
- **Consiglio:** durante l'installazione spuntare le 4 opzioni aggiuntive evidenziate (apertura di file e cartelle con VSCode dal menu contestuale, registrazione come editor di default, aggiunta a PATH)

### Docker

Docker è un **prerequisito obbligatorio** per Kathará.

|Sistema operativo|Installazione|
|---|---|
|**Windows**|Docker Desktop da https://docs.docker.com/desktop/setup/install/windows-install/|
|**Linux (Debian)**|Docker Engine dalla guida https://docs.docker.com/engine/install/|
|**macOS**|Docker Desktop da https://docs.docker.com/desktop/setup/install/mac-install/|

> ⚠️ Se Docker non è in esecuzione, Kathará non può partire.

### Kathará

- Disponibile per: Windows, Linux, macOS
- Download: https://www.kathara.org/download.html

> ⚠️ **Su Windows:** Windows Defender potrebbe bloccare l'installer segnalando che non è sicuro. Cliccare su **"Ulteriori informazioni"** → **"Esegui comunque"**. L'installer è legittimo.

---

## Avviare e verificare Kathará

**Procedura:**

1. Avviare Docker (es. aprire Docker Desktop)
2. Aprire un terminale (**PowerShell** su Windows)
3. Eseguire il comando di verifica:

```bash
kathara check
```

Se il comando non restituisce errori → Docker e Kathará sono installati e funzionanti correttamente.

---

## Struttura di un Kathará Lab

Un **Kathará lab** è un insieme di macchine virtuali preconfigurate che possono essere avviate e fermate insieme, simulando una topologia di rete.

Un lab è una **cartella** contenente:

```
my-lab/
├── lab.conf              ← topologia di rete
├── pc1.startup           ← comandi di avvio per pc1
├── pc2.startup           ← comandi di avvio per pc2
├── pc1/                  ← file copiati nel filesystem di pc1
│   └── etc/...
├── pc2/
└── shared/               ← directory condivisa tra host e tutti i device
```

### Il file lab.conf

Descrive la **topologia di rete**: i device presenti e come sono collegati tra loro.

**Sintassi:**

```
machine[arg]=value
```

|Elemento|Significato|
|---|---|
|`machine`|Nome del device (es. `pc1`)|
|`arg` = numero|L'interfaccia `eth<arg>` viene connessa al collision domain `value`|
|`arg` = stringa|È un'opzione di configurazione, `value` è il suo argomento|

**Esempio:**

```
pc1[0]=A
pc2[0]=A
pc2[1]=B
pc3[0]=B
```

Topologia risultante:

```
  pc1          pc2          pc3
  eth0         eth0  eth1   eth0
   |            |     |      |
   +----[ A ]---+     +--[B]-+
  
  Collision     Collision
  Domain A      Domain B
```

- `pc1` e `pc2` condividono il collision domain **A** (LAN A)
- `pc2` e `pc3` condividono il collision domain **B** (LAN B)
- `pc2` ha **due interfacce** → funge da router tra le due LAN

### I file .startup

Sono **shell script** eseguiti automaticamente da ogni device subito dopo l'avvio.

Uso tipico: configurare le interfacce di rete e avviare servizi.

**Esempio** (`pc1.startup`):

```bash
ip address add 10.0.0.1/24 dev eth0
systemctl start frr
```

### Condivisione di file tra host e device

Esistono due modalità:

#### File mirrored (sincronizzati in tempo reale)

|Directory nel device|Collegata a|Default|
|---|---|---|
|`/shared`|Cartella `shared/` dentro il lab|**ENABLED**|
|`/hosthome`|Home directory dell'utente host|DISABLED|

Le modifiche su un lato si riflettono automaticamente sull'altro.

#### File copiati (copie indipendenti)

Collocare i file nelle sottocartelle del lab con il nome del device:

```
my-lab/
└── pc1/
    └── prova/
        └── file.txt    →  copiato in /prova/file.txt dentro pc1
```

Le modifiche su una copia **non influenzano** l'altra.

---

## Comandi principali

|Comando|Azione|
|---|---|
|`kathara check`|Verifica installazione|
|`kathara lstart`|Avvia il lab (dalla cartella del lab)|
|`kathara lclean`|Ferma il lab|
|`kathara lrestart`|Riavvia il lab|

**Workflow standard:**

```bash
cd path/to/my-lab
kathara lstart
# ... lavoro ...
kathara lclean
```

---

## Indirizzi IP – Concetti base

Per comunicare su rete, ogni interfaccia di un device deve avere un **indirizzo IP** configurato:

```bash
ip address add 10.0.0.1/24 dev eth0
```

Caratteristiche degli indirizzi IPv4:

- Composti da **4 byte**, scritti come 4 numeri separati da `.`
- Ogni byte: valore da **0 a 255**
- Sono indirizzi di **Livello 3** (Network Layer) del modello ISO/OSI
- Il suffisso `/24` indica la **subnet mask** (verrà approfondito nel corso)

---

## Notazione grafica delle topologie

Nelle slide del corso viene usata la seguente convenzione:

```
        2              1        1              5
      [eth0]         [eth0]   [eth1]         [eth0]
        |               |       |               |
        +-------[A]-----+       +------[B]------+
                |                       |
           20.1.1.0/24             20.1.2.0/24
```

|Elemento|Significato|
|---|---|
|Lettera nel cerchio (`A`, `B`)|Nome del collision domain (LAN)|
|Numero nel box sopra l'interfaccia|**Ultimo byte** dell'indirizzo IP|
|Notazione sotto il bus (`20.1.1.0/24`)|**Prefisso IP** della LAN (i primi 3 byte sono uguali per tutti i device sulla stessa LAN)|

**Esempio di lettura:** un device con eth0 connessa ad A con numero `2` ha indirizzo IP `20.1.1.2/24`.

### Comando ping

```bash
ping <ip_address>
```

Testa la raggiungibilità di un altro host (lavora a Livello 3). Per ora usarlo **solo all'interno della stessa LAN** (stesso collision domain) — per uscire dalla LAN servono ulteriori configurazioni (routing, che verrà trattato più avanti).

---

## Domande tipiche da esame

**Q: Cos'è un Kathará lab?**  
Un insieme di macchine virtuali preconfigurate avviabili e fermabili insieme, descritte da una cartella contenente `lab.conf`, file `.startup` e sottocartelle per i device.

**Q: Cosa descrive il file `lab.conf`?**  
La topologia di rete: quali device esistono, a quali collision domain sono connesse le loro interfacce, e opzioni di configurazione aggiuntive.

**Q: Differenza tra file mirrored e file copiati?**  
I file mirrored (`/shared`, `/hosthome`) sono sincronizzati in tempo reale tra host e device. I file copiati (tramite sottocartella `<device>/`) sono copie indipendenti: le modifiche su una non si propagano all'altra.

**Q: Come si verifica che Kathará funzioni correttamente?**  
Avviare Docker, aprire un terminale ed eseguire `kathara check`. Nessun errore = installazione corretta.

**Q: Cosa succede se Docker non è in esecuzione quando si avvia Kathará?**  
Kathará non può partire. Docker deve essere avviato prima di qualsiasi operazione con Kathará.

**Q: Come si interpreta `pc2[1]=B` nel `lab.conf`?**  
L'interfaccia `eth1` di `pc2` viene connessa al collision domain `B`.