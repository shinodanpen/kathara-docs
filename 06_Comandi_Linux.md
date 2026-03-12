# Comandi e Configurazione Base dei Device Linux

← [01_INDEX](01_INDEX.md)

I device Katharà sono container Linux con strumenti di rete preinstallati (`iproute2`, `tcpdump`, `scapy`, ecc.).

> **Nota:** i comandi `ifconfig` e `route` (vecchio stile) potrebbero non essere disponibili. Usa sempre `ip` al loro posto.

---

## Interfacce di rete

### `ip addr` — Gestione indirizzi IP

```bash
ip addr [show [DEV]]         # mostra indirizzi (tutti, o solo DEV)
ip addr add IP/MASK dev DEV  # assegna un indirizzo all'interfaccia DEV
ip addr del IP/MASK dev DEV  # rimuove un indirizzo dall'interfaccia DEV
```

```bash
ip addr
ip addr show eth0
ip addr add 192.168.1.1/24 dev eth0
ip addr del 192.168.1.1/24 dev eth0
```

---

### `ip link` — Stato interfacce (layer 2)

```bash
ip link show [DEV]           # mostra stato interfacce (tutte, o solo DEV)
ip link set DEV up           # attiva l'interfaccia DEV
ip link set DEV down         # disattiva l'interfaccia DEV
```

```bash
ip link show
ip link show eth0
ip link set eth0 up
ip link set eth0 down
```

---

## Routing

### `ip route` — Gestione tabella di routing

```bash
ip route [show]                          # mostra la tabella di routing
ip route add NETWORK/MASK via GATEWAY    # aggiunge una route verso una rete
ip route add NETWORK/MASK dev DEV        # aggiunge una route su un'interfaccia diretta
ip route add default via GATEWAY         # aggiunge il default gateway
ip route del NETWORK/MASK [via GATEWAY]  # rimuove una route
```

```bash
ip route
ip route add 10.0.0.0/8 via 192.168.1.1
ip route add 10.0.0.0/8 dev eth1
ip route add default via 192.168.1.1
ip route del 10.0.0.0/8 via 192.168.1.1
```

> `ip route add default` e `ip route add 0.0.0.0/0` sono equivalenti.

Per una visualizzazione più leggibile della routing table, vedi [routel](07b_Router_Linux.md#routel).

---

## Test di connettività

### `ping` — Verifica raggiungibilità di un host

```bash
ping HOST                    # ping continuo verso HOST (CTRL+C per fermare)
ping -c COUNT HOST           # invia esattamente COUNT pacchetti
ping -i INTERVAL HOST        # imposta l'intervallo tra pacchetti in secondi
ping -W TIMEOUT HOST         # imposta il timeout per ogni risposta in secondi
```

```bash
ping 192.168.1.1
ping -c 4 192.168.1.1
ping -c 3 -i 0.5 10.0.0.1
```

---

### `traceroute` — Traccia il percorso verso un host

```bash
traceroute [-m MAX_HOPS] [-w TIMEOUT] HOST
```

Mostra tutti i router attraversati per raggiungere HOST, con i relativi tempi di risposta.

| Parametro | Descrizione |
|---|---|
| `-m MAX_HOPS` | Numero massimo di hop (default: 30) |
| `-w TIMEOUT` | Timeout in secondi per ogni risposta (default: 5) |

```bash
traceroute 10.0.0.2
traceroute -m 10 10.0.0.2
```

> `* * *` su una riga significa che quel router non ha risposto (potrebbe non supportare ICMP o avere il rate limiting attivo).

---

## Analisi del traffico

### `tcpdump` — Cattura e analisi pacchetti

```bash
tcpdump [-i DEV] [-c COUNT] [-n] [-v] [FILTRO]
```

Cattura e mostra in tempo reale i pacchetti che transitano su un'interfaccia. Senza opzioni ascolta su tutte le interfacce. Interrompi con CTRL+C.

| Parametro | Descrizione |
|---|---|
| `-i DEV` | Interfaccia su cui ascoltare (es. `eth0`, `any` per tutte) |
| `-c COUNT` | Cattura solo COUNT pacchetti poi si ferma |
| `-n` | Non risolve nomi DNS (mostra IP numerici — più veloce e leggibile in lab) |
| `-v` / `-vv` | Output più dettagliato |
| `FILTRO` | Espressione di filtro BPF |

```bash
tcpdump -i eth0                          # tutto il traffico su eth0
tcpdump -i eth0 -n                       # senza risoluzione DNS
tcpdump -i eth0 -n -c 20                 # cattura solo 20 pacchetti
tcpdump -i eth0 host 192.168.1.2         # solo traffico da/verso quell'IP
tcpdump -i eth0 src 192.168.1.2          # solo traffico proveniente da quell'IP
tcpdump -i eth0 dst 10.0.0.1             # solo traffico diretto a quell'IP
tcpdump -i eth0 port 80                  # solo traffico sulla porta 80
tcpdump -i eth0 icmp                     # solo pacchetti ICMP (ping)
tcpdump -i eth0 icmp and host 10.0.0.1   # ICMP da/verso un IP specifico
```

> **Filtri utili:** `icmp`, `tcp`, `udp`, `arp`, `host IP`, `port N`, `net NETWORK/MASK`. Si possono combinare con `and`, `or`, `not`.

Per un'analisi grafica del traffico, vedi [08_Wireshark](08_Wireshark.md).

---

## Risoluzione dei nomi

### `/etc/hosts` — Mappature statiche hostname → IP

```bash
cat /etc/hosts                           # mostra le mappature statiche
echo "IP HOSTNAME" >> /etc/hosts         # aggiunge una mappatura
```

```bash
cat /etc/hosts
echo "10.0.0.1 router1" >> /etc/hosts
```

---

## Processi e servizi

### `ps` — Lista processi in esecuzione

```bash
ps aux                                   # mostra tutti i processi con dettagli
ps aux | grep NOME                       # filtra per nome processo
```

---

### `ss` — Connessioni e porte in ascolto

```bash
ss -tuln                                 # porte TCP/UDP in ascolto (senza risoluzione DNS)
ss -tp                                   # connessioni TCP attive con PID
```

| Flag | Descrizione |
|---|---|
| `-t` | Connessioni TCP |
| `-u` | Connessioni UDP |
| `-l` | Solo porte in ascolto (listening) |
| `-n` | Mostra numeri invece di nomi |
| `-p` | Mostra il processo associato |

---

## Riepilogo rapido

| Comando | Cosa fa |
|---|---|
| `ip addr` | Mostra/gestisce indirizzi IP |
| `ip link set DEV up/down` | Attiva/disattiva un'interfaccia |
| `ip route` | Mostra/gestisce la tabella di routing |
| `ping [-c N] HOST` | Testa la raggiungibilità |
| `traceroute HOST` | Traccia il percorso verso un host |
| `tcpdump -i DEV` | Cattura traffico su un'interfaccia |
| `ss -tuln` | Mostra porte in ascolto |
