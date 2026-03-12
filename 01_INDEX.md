# Documentazione Katharà

Documentazione tecnica per l'utilizzo di Katharà, ottimizzata per consultazione da Obsidian.

---

## Sezioni

### Fondamentali
- [02_Premessa](02_Premessa.md) — Concetti base: device e collision domain

### Katharà
- [03_Comandi_Kathara](03_Comandi_Kathara.md) — Tutti i comandi con sintassi, opzioni ed esempi
- [04_Struttura_Laboratorio](04_Struttura_Laboratorio.md) — Organizzazione dei file, `lab.conf`, immagini Docker
- [05_Condivisione_File](05_Condivisione_File.md) — Scambiare file tra host e device

### Device Linux
- [06_Comandi_Linux](06_Comandi_Linux.md) — Comandi bash disponibili su tutti i device
- [07_Dispositivi_Rete](07_Dispositivi_Rete.md) — Configurazioni specifiche per dispositivo
  - [07a_Bridge_Linux](07a_Bridge_Linux.md) — Bridge software a livello 2
  - [07b_Router_Linux](07b_Router_Linux.md) — Router con routing statico

### Strumenti
- [08_Wireshark](08_Wireshark.md) — Wireshark in Katharà: configurazione, accesso, cattura

### Laboratori
- [09_Laboratori](09_Laboratori.md) — Analisi dei laboratori proposti

---

## Concetti chiave trasversali

| Concetto | Dove è definito |
|---|---|
| Device | [02_Premessa](02_Premessa.md#device) |
| Collision domain | [02_Premessa](02_Premessa.md#collision-domain) |
| `lab.conf` | [04_Struttura_Laboratorio](04_Struttura_Laboratorio.md#labconf) |
| Immagini Docker | [04_Struttura_Laboratorio](04_Struttura_Laboratorio.md#immagini-docker) |
| FDB | [07a_Bridge_Linux](07a_Bridge_Linux.md#filtering-database-fdb) |
| ARP | [07b_Router_Linux](07b_Router_Linux.md#arp-nei-router) |
| Routing table | [06_Comandi_Linux](06_Comandi_Linux.md#routing) |
