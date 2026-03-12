# Condivisione di File tra Host e Device

← [01_INDEX](01_INDEX.md)

Ci sono due modi per scambiare file tra il filesystem dell'host e quello dei device in Katharà.

---

## File mirrored (sincronizzati in tempo reale)

Le modifiche su un lato si riflettono automaticamente sull'altro. Ci sono due directory mirror disponibili:

| Directory nel device | Corrisponde a (sull'host) | Attiva per default? |
|---|---|---|
| `/shared` | cartella `shared/` nella cartella del lab | **sì** |
| `/hosthome` | home directory dell'utente sull'host | no |

**`/shared`** è la modalità più comoda per scambiare file durante un lab attivo: basta mettere o leggere file dalla cartella `shared/` nella directory del laboratorio sull'host.

**`/hosthome`** va abilitata esplicitamente con `--hosthome` in [lstart](03_Comandi_Kathara.md#lstart) / [vstart](03_Comandi_Kathara.md#vstart), oppure nelle impostazioni globali. Monta la home dell'utente host direttamente nel device.

> Per controllare il mount di `shared/` su singoli device, si può usare l'opzione `[shared]` in [lab.conf](04_Struttura_Laboratorio.md#labconf).

---

## File copiati (copie indipendenti)

I file messi nelle sottocartelle dei device nella cartella del lab vengono **copiati** dentro il device all'avvio. Le modifiche successive su un lato **non** si riflettono sull'altro.

La struttura delle sottocartelle viene mappata sulla root `/` del device:

```
lab/
└── pc1/
    └── prova/
        └── file.txt   →   copiato in /prova/file.txt dentro pc1
```

Stessa logica per file di configurazione di sistema:

```
lab/
└── pc1/
    └── etc/
        └── resolv.conf   →   copiato in /etc/resolv.conf dentro pc1
```

---

## Confronto tra i due metodi

| | File mirrored (`/shared`) | File copiati (sottocartelle) |
|---|---|---|
| Sincronizzazione | In tempo reale, bidirezionale | Solo all'avvio, unidirezionale |
| Uso tipico | Scambio dinamico durante il lab | Pre-caricare configurazioni prima dell'avvio |
| Persistenza dopo `lclean` | I file restano nella cartella `shared/` del lab | I file restano nella sottocartella del device |
