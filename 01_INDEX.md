# Documentazione Katharà

Documentazione tecnica per l'utilizzo di Katharà, ottimizzata per consultazione da Obsidian.

---

## Sezioni

### Fondamentali
- [[02_Premessa]] — Concetti base: device e collision domain

### Katharà
- [[03_Comandi_Kathara]] — Tutti i comandi con sintassi, opzioni ed esempi
- [[04_Struttura_Laboratorio]] — Organizzazione dei file, `lab.conf`, immagini Docker
- [[05_Condivisione_File]] — Scambiare file tra host e device

### Device Linux
- [[06_Comandi_Linux]] — Comandi bash disponibili su tutti i device
- [[07_Dispositivi_Rete]] — Configurazioni specifiche per dispositivo
  - [[07a_Bridge_Linux]] — Bridge software a livello 2
  - [[07b_Router_Linux]] — Router con routing statico

### Strumenti
- [[08_Wireshark]] — Wireshark in Katharà: configurazione, accesso, cattura

### Laboratori
- [[INDEX laboratori]] — Analisi dei laboratori proposti

---

## Concetti chiave trasversali

| Concetto | Dove è definito |
|---|---|
| Device | [[02_Premessa#device]] |
| Collision domain | [[02_Premessa#collision-domain]] |
| `lab.conf` | [[04_Struttura_Laboratorio#labconf]] |
| Immagini Docker | [[04_Struttura_Laboratorio#immagini-docker]] |
| FDB | [[07a_Bridge_Linux#filtering-database-fdb]] |
| ARP | [[07b_Router_Linux#arp-nei-router]] |
| Routing table | [[06_Comandi_Linux#routing]] |
