# Configurazioni Specifiche per Dispositivi di Rete

← [01_INDEX](01_INDEX.md)

Questa sezione raccoglie le configurazioni e i comandi specifici per ciascun tipo di dispositivo di rete emulabile in Katharà. I comandi Linux generali (interfacce, routing, ping, tcpdump...) sono in [06_Comandi_Linux](06_Comandi_Linux.md).

L'immagine base `kathara/base` è sufficiente per tutti i dispositivi qui descritti. Il comportamento dipende esclusivamente da ciò che si scrive nel file `.startup`. Vedi [immagini Docker](04_Struttura_Laboratorio.md#immagini-docker) per il dettaglio.

---

## Dispositivi disponibili

- [07a_Bridge_Linux](07a_Bridge_Linux.md) — Switch software a livello 2 (MAC address, FDB, flooding)
- [07b_Router_Linux](07b_Router_Linux.md) — Router a livello 3 (IP, routing table, ARP)

---

> Sezioni future da aggiungere man mano che vengono trattate nei laboratori:
> - Switch con VLAN
> - Firewall / NAT
> - Host con servizi (server HTTP, DNS, ecc.)
