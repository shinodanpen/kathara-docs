# Struttura di un Laboratorio

← [[01_INDEX]]

Uno **scenario di rete** è una cartella che contiene la descrizione completa di una rete virtuale: quali device esistono, come sono collegati, come si configurano all'avvio. Si lancia con `lstart` da dentro quella cartella (o passando il percorso con `-d`). Vedi [[03_Comandi_Kathara#lstart|kathara lstart]].

---

## Struttura della cartella

```
mio-laboratorio/
│
├── lab.conf              ← topologia e opzioni dei device (quasi sempre obbligatorio)
├── lab.dep               ← dipendenze di avvio tra device (opzionale)
├── lab.ext               ← collegamento a reti fisiche esterne via VLAN (opzionale, solo Linux root)
│
├── pc1/                  ← cartella del device "pc1" (opzionale)
│   └── etc/
│       └── resolv.conf   ← file copiato in /etc/resolv.conf dentro il device all'avvio
│
├── pc1.startup           ← script eseguito all'avvio di pc1 (opzionale)
├── pc1.shutdown          ← script eseguito allo spegnimento di pc1 (opzionale)
│
├── shared/               ← cartella condivisa tra host e tutti i device in /shared (opzionale)
├── shared.startup        ← script eseguito su TUTTI i device prima del loro .startup (opzionale)
└── shared.shutdown       ← script eseguito su TUTTI i device dopo il loro .shutdown (opzionale)
```

> **Regola delle sottocartelle device:** la cartella `pc1/` viene trattata come se fosse la root `/` del device. Un file in `pc1/etc/resolv.conf` finirà in `/etc/resolv.conf` dentro il container. Vedi anche [[05_Condivisione_File]].

---

## Quali file servono, quando

| File/Cartella | Obbligatorio? | Quando usarlo |
|---|---|---|
| `lab.conf` | Quasi sempre sì | Definisce la topologia. Senza, `lstart` non parte (a meno di `-F`). |
| `lab.dep` | No | Solo se certi device devono aspettare che altri siano già up. |
| `lab.ext` | No | Solo se vuoi connettere un collision domain a una rete fisica reale (Linux + root). |
| `device/` | No | Solo se devi pre-popolare il filesystem del device con file. |
| `device.startup` | No | Quasi sempre utile: configura IP, avvia demoni, ecc. |
| `device.shutdown` | No | Raramente necessario. |
| `shared/` | No | Utile per scambiare file tra host e device durante il lab. |
| `shared.startup` | No | Se hai operazioni comuni da fare su tutti i device all'avvio. |
| `shared.shutdown` | No | Se hai operazioni comuni da fare su tutti i device allo spegnimento. |

---

## `lab.conf` — il file più importante

Sintassi generale: `device[opzione]=valore`

### Collegare interfacce a collision domain

Il numero tra parentesi quadre indica l'interfaccia (`eth0`, `eth1`, ...). Il valore è il nome del [[02_Premessa#collision-domain|collision domain]], con MAC opzionale.

```
pc1[0]="A"                        # eth0 di pc1 → collision domain A
pc1[1]="B"                        # eth1 di pc1 → collision domain B
r1[0]="A"                         # eth0 di r1  → collision domain A
r1[1]="C/02:42:ac:11:00:02"       # eth1 di r1  → dominio C con MAC specifico
```

### Opzioni per device

Ogni opzione si scrive come `device[opzione]=valore` e si applica solo al device specificato.

| Opzione | Valore | Descrizione |
|---|---|---|
| `[image]` | `"nome:tag"` | Immagine Docker da usare. Default: `kathara/base`. Vedi [[04_Struttura_Laboratorio#immagini-docker]]. |
| `[ipv6]` | `"true"` / `"false"` | Abilita o disabilita IPv6. Default: `"true"`. Disabilitarlo evita il traffico NDP automatico che interferirebbe con scenari dove si vuole osservare il bridge learning passo dopo passo. |
| `[mem]` | `"128m"`, `"1g"`, ... | Limita la RAM del container. Minimo `4m`. |
| `[cpus]` | `0.5`, `1.5`, ... | Limita l'uso di CPU (valore float rispetto ai core disponibili). |
| `[shell]` | `"/bin/bash"` | Shell usata per i terminali e per eseguire i file `.startup`. |
| `[exec]` | `"comando"` | Comando aggiuntivo eseguito durante lo startup, dopo il file `.startup`. |
| `[bridged]` | `true` | Aggiunge un'interfaccia NAT verso la rete host, configurata via DHCP. Utile per accesso internet o per esporre servizi. |
| `[port]` | `"HOST:GUEST/PROTO"` | Mappa una porta host a una porta del device. `PROTO` può essere `tcp` o `udp` (default: `tcp`). |
| `[num_terms]` | `0`, `1`, `2`, ... | Numero di terminali da aprire all'avvio. `0` non apre nessun terminale. |
| `[sysctl]` | `"net.X.Y=valore"` | Imposta un parametro sysctl nel namespace `net.`. Usato tipicamente per abilitare il forwarding IP su [[07b_Router_Linux\|router]]. |
| `[privileged]` | `"true"` | Avvia il container in modalità privilegiata. Richiede Katharà avviato come root. |
| `[hosthome]` | `"true"` / `"false"` | Monta la home dell'utente host in `/hosthome` dentro il device. |
| `[shared]` | `"true"` / `"false"` | Monta la cartella `shared/` in `/shared` dentro il device. |
| `[env]` | `"NOME=VALORE"` | Imposta una variabile d'ambiente nel container. |
| `[ulimit]` | `"KEY=SOFT[:HARD]"` | Imposta un ulimit. Usa `-1` per illimitato. |

### Metadati dello scenario

Opzionali, vengono mostrati all'avvio del lab.

```
LAB_NAME="Esempio"
LAB_DESCRIPTION="Rete semplice con un router e due PC"
LAB_VERSION=1.0
LAB_AUTHOR="Mario"
```

---

## `lab.dep` — dipendenze di avvio

Garantisce che certi device vengano avviati solo dopo altri. Utile quando un device deve trovare già attivo un router o un server.

```
# pc1 parte solo dopo che pc2 e pc3 sono up
pc1: pc2 pc3
# pc3 parte solo dopo pc2
pc3: pc2
```

---

## `lab.ext` — reti esterne (avanzato)

Collega un [[02_Premessa#collision-domain|collision domain]] a un'interfaccia fisica dell'host, con supporto VLAN. Solo Linux, solo root. Da usare con cautela.

```
# Dominio A sull'interfaccia enp9s0 (no VLAN)
A enp9s0
# Dominio B su enp9s0 con VLAN ID 20
B enp9s0.20
# Dominio C su eth1 con VLAN ID 4001
C eth1.4001
```

> Quando `lab.ext` è presente, i terminali non si aprono automaticamente. Usare [[03_Comandi_Kathara#connect|kathara connect]] per accedere ai device.

---

## Immagini Docker

L'immagine standard è `kathara/base`: una macchina Linux minimale con strumenti di rete preinstallati (`iproute2`, `tcpdump`, `ping`, `scapy`, ecc.). Il comportamento del device dipende dai comandi nel `.startup`, non dall'immagine.

| Immagine | Uso tipico | Note |
|---|---|---|
| `kathara/base` | PC, [[07b_Router_Linux\|router]], [[07a_Bridge_Linux\|bridge]] | Immagine standard, usata per quasi tutto. |
| `kathara/frr` | Routing dinamico (OSPF, BGP, ecc.) | Contiene FRRouting preinstallato. |
| `lscr.io/linuxserver/wireshark` | Analisi del traffico via browser | Vedi [[08_Wireshark]]. |

> Il comportamento da router o bridge si ottiene **interamente** tramite i comandi nel file `.startup`, non dall'immagine. `kathara/base` è sufficiente per entrambi.

---

## Esempio completo: rete con un router e due PC

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

```bash
ip addr add 10.0.0.2/24 dev eth0
ip route add default via 10.0.0.1
```

Per la configurazione dettagliata delle interfacce e del routing, vedi [[06_Comandi_Linux]] e [[07b_Router_Linux]].
