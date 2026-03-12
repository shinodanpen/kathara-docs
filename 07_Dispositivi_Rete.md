# Configurazioni Specifiche per Dispositivi di Rete

← [[01_INDEX]]

Questa sezione raccoglie le configurazioni e i comandi specifici per ciascun tipo di dispositivo di rete emulabile in Katharà. I comandi Linux generali (interfacce, routing, ping, tcpdump...) sono in [[06_Comandi_Linux]].

L'immagine base `kathara/base` è sufficiente per tutti i dispositivi qui descritti. Il comportamento dipende esclusivamente da ciò che si scrive nel file `.startup`. Vedi [[04_Struttura_Laboratorio#immagini-docker|immagini Docker]] per il dettaglio.

---

## Dispositivi disponibili

- [[07a_Bridge_Linux]] — Switch software a livello 2 (MAC address, FDB, flooding)
- [[07b_Router_Linux]] — Router a livello 3 (IP, routing table, ARP)

---

> Sezioni future da aggiungere man mano che vengono trattate nei laboratori:
> - Switch con VLAN
> - Firewall / NAT
> - Host con servizi (server HTTP, DNS, ecc.)
