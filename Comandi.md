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

| Parametro     | Descrizione                                       |
| ------------- | ------------------------------------------------- |
| `-m MAX_HOPS` | Numero massimo di hop (default: 30)               |
| `-w TIMEOUT`  | Timeout in secondi per ogni risposta (default: 5) |

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

| Parametro    | Descrizione                                                               |
| ------------ | ------------------------------------------------------------------------- |
| `-i DEV`     | Interfaccia su cui ascoltare (es. `eth0`, `any` per tutte)                |
| `-c COUNT`   | Cattura solo COUNT pacchetti poi si ferma                                 |
| `-n`         | Non risolve nomi DNS (mostra IP numerici — più veloce e leggibile in lab) |
| `-v` / `-vv` | Output più dettagliato                                                    |
| `FILTRO`     | Espressione di filtro BPF (vedi esempi)                                   |

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

| Comando                                           | Cosa fa                               |
| ------------------------------------------------- | ------------------------------------- |
| `ip addr`                                         | Mostra/gestisce indirizzi IP          |
| `ip link set DEV up/down`                         | Attiva/disattiva un'interfaccia       |
| `ip route`                                        | Mostra/gestisce la tabella di routing |
| `ping [-c COUNT] [-i INTERVAL] [-W TIMEOUT] HOST` | Testa la raggiungibilità              |
| `traceroute HOST`                                 | Traccia il percorso verso un host     |
| `tcpdump -i DEV`                                  | Cattura traffico su un'interfaccia    |
| `ss -tuln`                                        | Mostra porte in ascolto               |