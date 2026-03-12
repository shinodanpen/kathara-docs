# Comandi e Parametri di Katharà

← [[01_INDEX]]

Katharà è un emulatore di reti basato su container Docker. Ogni dispositivo è un container; ogni collegamento è una rete virtuale chiamata [[02_Premessa#collision-domain|collision domain]]. È il successore spirituale di Netkit, con cui è cross-compatibile.

I comandi si dividono in tre famiglie:

- **v-comandi** (`vstart`, `vclean`, `vconfig`): gestione di singoli device
- **l-comandi** (`lstart`, `lclean`, `linfo`, `lrestart`, `lconfig`): gestione di scenari di rete completi (vedi [[04_Struttura_Laboratorio]])
- **comandi globali** (`connect`, `exec`, `wipe`, `list`, `settings`, `check`)

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

Avvia un nuovo device con il nome specificato. I nomi devono essere unici per utente. Senza opzioni usa i default di `kathara.conf`.

| Opzione | Descrizione |
|---|---|
| `-n NOME` | Nome del device (obbligatorio) |
| `--eth N:CD[/MAC]` | Aggiunge un'interfaccia `eth_N_` collegata al [[02_Premessa#collision-domain|collision domain]] `CD`. MAC opzionale nel formato `XX:XX:XX:XX:XX:XX`. N deve essere sequenziale a partire da 0. |
| `--mem MEM` | Limita la RAM (es. `128m`, `1g`). Minimo 4m. |
| `--cpus N` | Limita CPU (float; es. `1.5` su host con 2 CPU). |
| `-i IMAGE` | Usa un'immagine Docker specifica. Vedi [[04_Struttura_Laboratorio#immagini-docker]]. |
| `-e COMANDO` | Esegue un comando dentro il device durante lo startup. |
| `--bridged` | Aggiunge un'interfaccia collegata alla rete host via NAT (configurata automaticamente via DHCP). Sarà l'ultima interfaccia. |
| `--port [HOST:]GUEST[/PROTO]` | Mappa una porta host a una porta del device. Default HOST: 3000, default PROTO: tcp. |
| `--hosthome` / `--no-hosthome` | Monta (o non monta) la home dell'utente dentro il device in `/hosthome`. Vedi [[05_Condivisione_File]]. |
| `--privileged` | Avvia il device in modalità privilegiata (richiede root). |
| `--noterminals` / `--terminals` | Controlla se aprire o meno finestre terminale. |
| `--num_terms N` | Numero di terminali da aprire. |
| `--shell SHELL` | Shell da usare dentro il device (es. `bash`, `sh`). |
| `--xterm APP` | Emulatore di terminale alternativo (solo Unix). |
| `--sysctl OPT` | Imposta opzioni sysctl (solo namespace `net.`). |
| `--env NOME=VALORE` | Imposta variabili d'ambiente. |
| `--ulimit KEY=SOFT[:HARD]` | Imposta ulimit. Usa `-1` per illimitato. |
| `--print` | Dry run: verifica i parametri senza avviare nulla. |

**Esempi:**

```bash
# Device semplice
kathara vstart -n pc1

# Device con RAM limitata e immagine custom
kathara vstart -n mypc1 -i my-image --mem 128m

# Due device collegati sullo stesso collision domain "A"
kathara vstart -n producer --eth 0:A
kathara vstart -n consumer --eth 0:A

# Device con eth0 su dominio A e interfaccia bridged verso la rete host
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

```bash
kathara vclean -n pc1
```

---

### `kathara vconfig` — Gestire interfacce di un device in esecuzione

```bash
kathara vconfig -n DEVICE_NAME (--add <CD[/MAC]> ... | --rm CD ...)
```

Aggiunge o rimuove interfacce di rete da un device già avviato. Il numero della nuova interfaccia viene assegnato automaticamente.

| Opzione | Descrizione |
|---|---|
| `--add CD[/MAC]` | Collega il device al [[02_Premessa#collision-domain|collision domain]] `CD` (MAC opzionale). |
| `--rm CD` | Scollega il device dal collision domain `CD` e rimuove l'interfaccia. |

```bash
kathara vconfig -n pc1 --add X Y
kathara vconfig -n pc1 --add X/00:00:00:00:00:01
kathara vconfig -n pc1 --rm X
```

---

## l-comandi (scenari di rete)

Uno **scenario di rete** è una cartella con file di configurazione e sottocartelle per ogni device. Vedi [[04_Struttura_Laboratorio]] per la struttura completa.

---

### `kathara lstart` — Avviare uno scenario

```bash
kathara lstart [opzioni] [DEVICE_NAME ...]
```

Avvia tutti i device dello scenario. Se si specificano dei nomi, avvia solo quelli.

| Opzione | Descrizione |
|---|---|
| `-d DIRECTORY` | Cartella dello scenario (default: directory corrente). |
| `-F` / `--force-lab` | Forza l'avvio anche senza `lab.conf`. |
| `-l` / `--list` | Mostra info sui device dopo l'avvio. |
| `-o "OPT" ...` | Applica opzioni (stile `lab.conf`) a tutti i device. |
| `--exclude NOME ...` | Esclude device specifici dall'avvio. |
| `--hosthome` / `--no-hosthome` | Monta/non monta `/hosthome` nei device. |
| `--shared` / `--no-shared` | Monta/non monta la cartella `shared/`. Vedi [[05_Condivisione_File]]. |
| `--noterminals` / `--terminals` | Controlla apertura terminali. |
| `--privileged` | Avvia in modalità privilegiata (richiede root). |
| `--print` | Dry run: verifica `lab.conf` senza avviare. |

---

### `kathara lclean` — Fermare uno scenario

```bash
kathara lclean [opzioni] [DEVICE_NAME ...]
```

Spegne tutti i device dello scenario (o solo quelli specificati).

| Opzione | Descrizione |
|---|---|
| `-d DIRECTORY` | Cartella dello scenario. |
| `--exclude NOME ...` | Esclude device specifici dallo spegnimento. |

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

Mostra lo stato di tutti i device dello scenario corrente. Colonne: `LAB_HASH`, `DEVICE_NAME`, `STATUS`, `CPU %`, `MEM USAGE / LIMIT`, `MEM %`, `NET I/O`.

| Opzione | Descrizione |
|---|---|
| `-d DIRECTORY` | Cartella dello scenario. |
| `-w` / `--watch` | Aggiornamento live (uscita con CTRL+C). |
| `-c` / `--conf` | Mostra info statiche da `lab.conf`. |
| `-t` / `--topology` | Mostra la mappatura device ↔ collision domain. |
| `-n NOME` | Filtra le info su un solo device. |

---

### `kathara lconfig` — Gestire interfacce in uno scenario

```bash
kathara lconfig [-d DIRECTORY] -n DEVICE_NAME (--add <CD[/MAC]> ... | --rm CD ...)
```

Identico a `vconfig`, ma opera su device appartenenti a uno scenario. Richiede `-n` per specificare il device target.

```bash
kathara lconfig -d path/to/scenario -n pc1 --add X Y
kathara lconfig -d path/to/scenario -n pc1 --add X/00:00:00:00:00:01
kathara lconfig -d path/to/scenario -n pc1 --rm X
```

> Usato anche per collegare [[08_Wireshark]] ai collision domain dopo l'avvio del lab.

---

## Comandi globali

### `kathara connect` — Aprire una shell su un device

```bash
kathara connect [-d DIRECTORY | -v] [--command SHELL] [-l] DEVICE_NAME
```

Apre una shell interattiva dentro il device specificato.

| Opzione | Descrizione |
|---|---|
| `-d DIRECTORY` | Device appartiene a uno scenario in DIRECTORY. |
| `-v` / `--vdevice` | Device avviato con `vstart`. Non combinabile con `-d`. |
| `--command SHELL` | Comando da eseguire (default: shell da `kathara.conf`). |
| `-l` / `--logs` | Stampa i log di startup prima di aprire la shell. |

```bash
kathara connect -v pc1        # device avviato con vstart
kathara connect as1r1         # device in uno scenario nella directory corrente
```

---

### `kathara exec` — Eseguire un comando in un device

```bash
kathara exec [-d DIRECTORY | -v] [--no-stdout] [--no-stderr] [--wait] DEVICE_NAME COMANDO
```

Esegue un comando dentro un device senza aprire una shell interattiva.

| Opzione | Descrizione |
|---|---|
| `-d DIRECTORY` | Device in uno scenario in DIRECTORY. |
| `-v` / `--vmachine` | Device avviato con `vstart`. |
| `--no-stdout` | Disabilita stdout del comando. |
| `--no-stderr` | Disabilita stderr del comando. |
| `--wait` | Aspetta che i comandi di startup finiscano. |

```bash
kathara exec -v pc1 -- ping 127.0.0.1
kathara exec as1r1 "ping 127.0.0.1"
```

---

### `kathara wipe` — Pulizia totale

```bash
kathara wipe [-f] [-s | -a]
```

Spegne **tutti** i device Katharà dell'utente corrente.

| Opzione | Descrizione |
|---|---|
| `-f` / `--force` | Non chiede conferma. |
| `-s` / `--settings` | Elimina e ricrea `kathara.conf` con valori default. Non compatibile con `-a`. |
| `-a` / `--all` | Pulisce i device di tutti gli utenti (richiede root). Non compatibile con `-s`. |

```bash
kathara wipe -s    # resetta solo le impostazioni
```

---

### `kathara list` — Elenco device in esecuzione

```bash
kathara list [-a] [-w] [-n DEVICE_NAME]
```

Mostra tutti i device in esecuzione dell'utente corrente.

| Opzione | Descrizione |
|---|---|
| `-a` / `--all` | Mostra device di tutti gli utenti (richiede root). |
| `-w` / `--watch` | Aggiornamento live (CTRL+C per uscire). |
| `-n NOME` | Filtra su un solo device. |

---

### `kathara settings` — Configurazione

```bash
kathara settings
```

Apre un'interfaccia console per visualizzare e modificare tutte le impostazioni in `~/.config/kathara.conf`.

---

### `kathara check` — Verifica ambiente

```bash
kathara check
```

Esegue una serie di test per verificare che l'ambiente sia configurato correttamente. Stampa le versioni di manager, Python, Katharà e OS.

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

| Comando | Cosa fa |
|---|---|
| `vstart -n NOME` | Avvia un device singolo |
| `vclean -n NOME` | Ferma un device singolo |
| `vconfig -n NOME --add/--rm` | Aggiunge/rimuove interfacce a device singolo |
| `lstart` | Avvia l'intero scenario |
| `lclean` | Ferma l'intero scenario |
| `lrestart` | Riavvia lo scenario |
| `linfo` | Mostra stato dello scenario |
| `lconfig -n NOME --add/--rm` | Aggiunge/rimuove interfacce in uno scenario |
| `connect NOME` | Apre shell su un device |
| `exec NOME COMANDO` | Esegue comando su un device |
| `wipe` | Spegne tutto |
| `list` | Elenca tutti i device attivi |
| `settings` | Modifica configurazione |
| `check` | Verifica l'ambiente |
