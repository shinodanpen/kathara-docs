Ci sono due modi per scambiare file tra il filesystem dell'host e quello dei device.

---

### File mirrored (sincronizzati in tempo reale)

Le modifiche su un lato si riflettono automaticamente sull'altro. Ci sono due directory mirror disponibili:

|Directory nel device|Corrisponde a|Default|
|---|---|---|
|`/shared`|cartella `shared/` nella cartella del lab|**abilitata**|
|`/hosthome`|home directory dell'utente sull'host|disabilitata|

`/shared` è la modalità più comoda per scambiare file durante un lab attivo — basta mettere o leggere file da `shared/` nella cartella del lab sull'host.

`/hosthome` va abilitata esplicitamente con `--hosthome` in `lstart`/`vstart` o nelle impostazioni.

---

### File copiati (copie indipendenti)

I file messi nelle sottocartelle dei device nella cartella del lab vengono **copiati** dentro il device all'avvio. Le modifiche successive su un lato non si riflettono sull'altro.

La struttura delle sottocartelle viene mappata sulla root `/` del device:

```
lab/
└── pc1/
    └── prova/
        └── file.txt   →   copiato in /prova/file.txt dentro pc1
```

> Questa modalità è utile per pre-caricare file di configurazione prima dell'avvio, mentre `/shared` è più adatta allo scambio dinamico durante il lab.