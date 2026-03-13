# Analisi dei Laboratori

← [01_INDEX](01_INDEX.md)

Questa sezione raccoglie l'analisi dettagliata dei laboratori proposti nelle slide del corso. Per ogni laboratorio: topologia, concetti chiave, cosa osservare, cosa imparare.

---

## Laboratori

| File                          | Titolo                                         | Concetti principali                                                              |
| ----------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------- |
| [Lab 1](09a_Laboratorio_1.md) | Installazione ambiente, primo lab, ping su LAN | Device, collision domain, lab.conf, .startup, ping                               |
| [Lab 2](09b_Laboratorio_2.md) | One Bridge: bridge learning e flooding         | Bridge learning, FDB, flooding, forwarding selettivo, scapy, Wireshark           |
| [Lab 3](09c_Laboratorio_3.md) | Basic IPv4, ping, traceroute e ARP             | Indirizzamento IPv4, default gateway, routing statico, ARP cache, traceroute TTL |
| [Lab 4](09d_Laboratorio_4.md) | Basic IPv6                                     | Indirizzamento IPv6, SLAAC, radvd, ICMPv6, NDP, neighbor cache                   |

---

## Come è strutturata ogni pagina di laboratorio

Ogni laboratorio avrà:

- **Topologia** — schema della rete, device coinvolti, collision domain
- **File di configurazione** — `lab.conf`, file `.startup` di ogni device
- **Concetti chiave** — cosa si studia in questo lab, con link alle sezioni teoriche
- **Cosa osservare** — comandi da eseguire durante il lab e output atteso
- **Cosa imparare** — takeaway principali

---

## Concetti trasversali

I laboratori faranno riferimento alle seguenti sezioni della documentazione:

- [02_Premessa](02_Premessa.md) — Device e collision domain
- [04_Struttura_Laboratorio](04_Struttura_Laboratorio.md) — Come è organizzato un lab
- [06_Comandi_Linux](06_Comandi_Linux.md) — Comandi da usare durante l'esecuzione
- [07a_Bridge_Linux](07a_Bridge_Linux.md) — Lab su bridge e switching
- [07b_Router_Linux](07b_Router_Linux.md) — Lab su routing statico
- [08_Wireshark](08_Wireshark.md) — Cattura e analisi del traffico
