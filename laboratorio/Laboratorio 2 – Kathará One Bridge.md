**Corso di Reti di Calcolatori** | Lezione del 19 febbraio 2026
## 1. Topologia di rete

La rete del laboratorio è composta da **4 PC** e **1 Bridge (b1)**, tutti collegati in un'unica LAN suddivisa in **4 domini di collisione** (A, B, C, D).

```
       [PC1]          [PC2]          [PC3]          [PC4]
   MAC: 00:00:00:00:00:01   :02          :03          :04
        eth0 (-)    eth0 (-)    eth0 (-)    eth0 (-)
          |            |            |            |
   CD: [ A ]        [ B ]        [ C ]        [ D ]
          |            |            |            |
        eth0         eth1         eth2         eth3
   MAC: 00:00:00:00:00:b1  :b2          :b3          :b4
                        [  B1 – Bridge  ]
```

**Punti fondamentali sulla topologia:**

- Non ci sono indirizzi IP configurati: si lavora **solo a livello 2 (MAC)**
- L'intera rete è **un'unica LAN**, divisa in domini di collisione dal Bridge
- Ogni interfaccia ha un MAC address semplice e riconoscibile, assegnato manualmente
- IPv6 è **disabilitato** su tutti i dispositivi (vedi sezione lab.conf)

**Perché il Bridge divide la LAN in domini di collisione?** Il Bridge utilizza la logica **store-and-forward**: riceve un pacchetto su un'interfaccia, lo salva, consulta il proprio Filtering Database (FDB) e lo ri-trasmette solo sull'interfaccia di destinazione corretta. Questo fa sì che un pacchetto su dominio A non collida mai con uno su dominio B.

---

## 2. File di configurazione: lab.conf

Il file `lab.conf` descrive la topologia della rete a Kathará.

```
# PC 1, 2, 3, 4
pc1[0]="A/00:00:00:00:00:01"
pc1[image]="kathara/base"
pc1[ipv6]="false"

pc2[0]="B/00:00:00:00:00:02"
pc2[image]="kathara/base"
pc2[ipv6]="false"

pc3[0]="C/00:00:00:00:00:03"
pc3[image]="kathara/base"
pc3[ipv6]="false"

pc4[0]="D/00:00:00:00:00:04"
pc4[image]="kathara/base"
pc4[ipv6]="false"

# Bridge b1 (4 interfacce, uno per dominio di collisione)
b1[0]="A/00:00:00:00:00:b1"
b1[1]="B/00:00:00:00:00:b2"
b1[2]="C/00:00:00:00:00:b3"
b1[3]="D/00:00:00:00:00:b4"
b1[image]="kathara/base"
b1[ipv6]="false"

# Macchina Wireshark (aggiunta in fondo)
wireshark[bridged]=true
wireshark[port]="3000:3000"
wireshark[image]="lscr.io/linuxserver/wireshark"
wireshark[num_terms]=0
```

### Spiegazione dei parametri

|Parametro|Significato|
|---|---|
|`device[N]="CD/MAC"`|Collega la porta N del device al dominio di collisione CD con il MAC specificato|
|`[image]="kathara/base"`|Usa un'immagine Linux generica (non hardware specifico)|
|`[ipv6]="false"`|Disabilita IPv6 sulla macchina|
|`[bridged]=true`|Espone un'interfaccia raggiungibile dall'esterno del lab|
|`[port]="3000:3000"`|Mappa la porta TCP 3000 del container sulla porta 3000 dell'host|
|`[num_terms]=0`|Non aprire il terminale della macchina all'avvio|

**Perché disabilitare IPv6?** Quando IPv6 è abilitato, le schede di rete si auto-assegnano un indirizzo IPv6 e cominciano immediatamente a scambiare pacchetti di discovery (Neighbor Discovery Protocol). Questi pacchetti "sporcherebbero" il FDB del Bridge prima che noi lo osserviamo passo passo. Disabilitando IPv6, il Bridge parte con il FDB vuoto e il learning avviene solo quando noi lo provochiamo.

---

## 3. Configurazione del Bridge: b1.startup

Il file `b1.startup` contiene i comandi eseguiti automaticamente all'avvio del container b1. Poiché l'immagine `kathara/base` è un sistema Linux generico, dobbiamo configurarlo manualmente come Bridge usando strumenti Linux.

```bash
# 1. Creare il bridge software
ip link add name mainbridge type bridge

# 2. Connettere ("enslaving") le 4 interfacce al bridge
ip link set dev eth0 master mainbridge
ip link set dev eth1 master mainbridge
ip link set dev eth2 master mainbridge
ip link set dev eth3 master mainbridge

# 3. Attivare il bridge (di default è DOWN)
ip link set up dev mainbridge

# 4. Impostare l'ageing time a 600 secondi (10 minuti)
brctl setageing mainbridge 600
```

### Spiegazione di ogni comando

**`ip link add name mainbridge type bridge`** Crea un oggetto software di tipo Bridge chiamato `mainbridge`. Nota: `b1` è il nome del container per Kathará; `mainbridge` è il nome interno del software bridge sulla macchina.

**`ip link set dev ethX master mainbridge`** Connette l'interfaccia `ethX` al bridge. Questo processo è detto **"enslaving"**: il bridge diventa il "master" (gestore) dell'interfaccia. Va ripetuto per tutte e 4 le interfacce (eth0–eth3).

**`ip link set up dev mainbridge`** Attiva il bridge. Finché questo comando non viene eseguito, i pacchetti ricevuti vengono ignorati e non viene aggiornato il FDB. Questo comando **avvia il processo di learning**.

**`brctl setageing mainbridge 600`** Imposta l'ageing time a 600 secondi (default: 300 s). Dopo questo tempo, una riga nel FDB che non viene "rinfrescata" da nuovi pacchetti scade e viene rimossa.

---

## 4. Comandi Kathará essenziali

|Comando|Descrizione|
|---|---|
|`kathara lstart`|Avvia il laboratorio (eseguire nella cartella con lab.conf)|
|`kathara lclean` o `kathara wipe -f`|Ferma e pulisce il laboratorio|
|`kathara connect <device>`|Si connette al terminale di un device (es. `kathara connect pc1`)|
|`kathara lconfig -n <device> --add <CD>`|Aggiunge un'interfaccia al device sul dominio di collisione specificato|

**Importante:** ogni laboratorio deve risiedere in una propria cartella separata. Se il Lab precedente non è stato fermato, usare `kathara wipe -f` prima di avviare quello nuovo.

### Comando chiave: aggiungere Wireshark a un dominio di collisione

```bash
# Dalla PowerShell/terminale HOST (non dentro il container)
kathara lconfig -n wireshark --add B
```

Questo aggiunge un'interfaccia (`eth1`, `eth2`, ... in ordine) alla macchina wireshark, connessa al dominio di collisione B. Per collegare al dominio C: `--add C`, ecc.

---

## 5. Wireshark su Kathará

### Accesso

Dopo aver avviato il lab, aprire un browser e navigare a:

```
localhost:3000
```

Questo apre l'interfaccia grafica di Wireshark in esecuzione nel container.

### Interfacce disponibili

- **`eth0`**: interfaccia "bridged" – afaccia sul PC host, non sui domini di collisione del lab
- **`eth1`, `eth2`, ...**: interfacce aggiunte manualmente con `kathara lconfig`, che affacciano sui vari domini di collisione

### Utilizzo

1. Doppio click sull'interfaccia desiderata per iniziare la cattura
2. Premere F5 o usare _Capture → Refresh Interfaces_ per vedere nuove interfacce aggiunte
3. Per fermare la cattura: pulsante rosso in alto a sinistra
4. Per chiudere: _File → Close_ (selezionare "Don't Save")

### Cosa vediamo in Wireshark

Ogni riga corrisponde a un pacchetto catturato. Le colonne principali:

- **Source / Destination**: indirizzi MAC (livello 2)
- **Protocol**: tipo di protocollo
- **Length**: lunghezza del frame
- **Info**: dettagli sul pacchetto

**Limitazione importante:** Wireshark cattura solo i pacchetti che raggiungono fisicamente l'interfaccia monitorata. Se il Bridge ha già imparato dove sta il destinatario e non fa flooding, il pacchetto non passerà su tutti i domini di collisione e quindi non sarà visibile su Wireshark attestato su un altro dominio.

---

## 6. Scapy: creare pacchetti manualmente

Scapy è uno strumento Python che permette di costruire e inviare pacchetti a basso livello. Nei terminali dei PC (via Kathará) si usa in modalità interattiva.

```python
# Avviare scapy sul terminale del PC
root@pc1:~$ scapy

# Creare un pacchetto Ethernet (livello 2)
>>> p = Ether(dst='00:00:00:00:00:02', src='00:00:00:00:00:01')

# Inviare il pacchetto sull'interfaccia eth0
>>> sendp(p, iface='eth0')
Sent 1 packets.
```

|Parametro|Significato|
|---|---|
|`Ether(...)`|Crea un frame Ethernet di livello 2|
|`dst=`|MAC address del destinatario|
|`src=`|MAC address del mittente|
|`sendp(p, iface='eth0')`|Invia il pacchetto sull'interfaccia specificata|

**Nota:** il pacchetto creato contiene solo l'header Ethernet (MAC source + MAC destination) senza payload significativo. È sufficiente per osservare il comportamento del Bridge.

Per inviare nuovamente lo stesso pacchetto (se la variabile `p` è già definita):

```python
>>> sendp(p, iface='eth0')
```

---

## 7. Il processo di Learning del Bridge

### Il Filtering Database (FDB)

Il Bridge mantiene una tabella chiamata **Filtering Database (FDB)** o **MAC address table**. Per visualizzarla:

```bash
# Sul terminale di b1
root@b1:~$ brctl showmacs mainbridge
```

Output tipico:

```
port no   mac addr            is local?   ageing timer
1         00:00:00:00:00:b1   yes         0.00
1         00:00:00:00:00:b1   yes         0.00
2         00:00:00:00:00:b2   yes         0.00
2         00:00:00:00:00:b2   yes         0.00
3         00:00:00:00:00:b3   yes         0.00
3         00:00:00:00:00:b3   yes         0.00
4         00:00:00:00:00:b4   yes         0.00
4         00:00:00:00:00:b4   yes         0.00
```

### Spiegazione delle colonne

|Colonna|Significato|
|---|---|
|`port no`|Numero di porta virtuale del bridge (≠ nome interfaccia! numerato nell'ordine di enslaving)|
|`mac addr`|Indirizzo MAC appreso|
|`is local?`|`yes` = MAC di un'interfaccia del bridge stesso; `no` = MAC appreso da un pacchetto ricevuto|
|`ageing timer`|Secondi da quando il MAC è stato visto l'ultima volta (0 per i local = non scadono mai)|

**Perché i MAC sono duplicati?** È un artificio del bridge software in Kathará: quando si esegue lo enslaving, il software vede le interfacce due volte (una volta come interfaccia del container, una come interfaccia del bridge). Non ha alcun effetto pratico.

**Perché `port no` ha numeri e non nomi ETH?** Il bridge software assegna numeri di porta propri, nell'ordine in cui vengono effettuati gli enslaving nel file .startup. Quindi la porta 1 corrisponde a eth0 (il primo enslaved), porta 2 a eth1, ecc.

### Regole del Learning

1. **Il Bridge impara solo il MAC del mittente** di ogni pacchetto ricevuto, non del destinatario
2. **Ogni volta che arriva un pacchetto dallo stesso mittente, il timer si azzera** (ageing timer reset)
3. **I MAC locali (is local = yes) non scadono mai** (ageing timer = 0.00 fisso)
4. **I MAC appresi (is local = no) scadono** dopo l'ageing time impostato (600 secondi in questo lab)

### Comportamento in base al FDB

|Situazione|Comportamento del Bridge|
|---|---|
|MAC destinatario **non** presente nel FDB|Flooding: il pacchetto viene inviato su **tutte** le porte tranne quella di arrivo|
|MAC destinatario **presente** nel FDB|Il pacchetto viene inoltrato **solo** sulla porta corrispondente|
|MAC sorgente non ancora nel FDB|Il Bridge **aggiunge** il MAC sorgente al FDB sulla porta di arrivo|

---

## 8. Scenario completo: osservazione step by step

Il professore ha proposto questo scenario per osservare il learning:

|Evento|Dominio osservato da Wireshark|Wireshark vede il pacchetto?|
|---|---|---|
|pc1 → pc2|B|**Sì** (flooding, FDB vuoto)|
|pc1 → pc2|C|**Sì** (flooding, pc2 ancora non nel FDB)|
|pc2 → pc1|C|**No** (b1 conosce già pc1 su porta 1, inoltra solo su A)|
|pc1 → pc2|C|**No** (b1 ora conosce pc2 su porta 2, inoltra solo su B)|
|pc1 → pc4|C|**Sì** (b1 non conosce pc4, flooding)|
|_(attesa > 10 min)_|—|FDB entries per pc1 e pc2 **scadono**|
|pc4 → pc2|C|**Sì** (b1 non conosce più pc2 dopo la scadenza, flooding)|

### Analisi dettagliata evento per evento

**Evento 1: pc1 → pc2, Wireshark su dominio B**

- FDB inizialmente contiene solo i MAC locali del bridge
- b1 riceve il pacchetto da pc1 su eth0 (dominio A)
- b1 **non conosce** dove sta pc2 → fa **flooding** su eth1, eth2, eth3 (domini B, C, D)
- Wireshark su B **vede** il pacchetto ✓
- Dopo questo evento: b1 aggiunge `00:00:00:00:00:01 → porta 1` nel FDB

**Evento 2: pc1 → pc2, Wireshark su dominio C**

- b1 conosce pc1 (porta 1) ma **non ancora** pc2
- Nuovo pacchetto da pc1 → flooding su tutti i domini
- Wireshark su C **vede** il pacchetto ✓
- L'ageing timer di pc1 si azzera (refresh)

**Evento 3: pc2 → pc1, Wireshark su dominio C**

- pc2 invia un pacchetto a pc1; arriva a b1 su eth1 (porta 2)
- b1 **conosce pc1** (porta 1) → manda il pacchetto **solo su eth0** (dominio A)
- Wireshark su C **non vede** nulla ✗
- Dopo questo evento: b1 aggiunge `00:00:00:00:00:02 → porta 2` nel FDB

**Evento 4: pc1 → pc2, Wireshark su dominio C**

- b1 ora conosce sia pc1 che pc2
- Il pacchetto viene inoltrato **solo su eth1** (dominio B, dove sta pc2)
- Wireshark su C **non vede** nulla ✗

**Evento 5: pc1 → pc4, Wireshark su dominio C**

- b1 **non conosce** pc4 (non ha mai mandato nulla)
- Flooding su tutti i domini → Wireshark su C **vede** il pacchetto ✓

**Evento 6: attesa > 10 minuti, poi pc4 → pc2**

- Dopo 600 secondi, le righe di pc1 e pc2 nel FDB scadono e vengono rimosse
- b1 non conosce più dove sta pc2
- Flooding → Wireshark su C **vede** il pacchetto ✓

---

## 9. Riepilogo concetti chiave

### Bridge vs Hub

|Caratteristica|Hub|Bridge|
|---|---|---|
|Livello OSI|1 (fisico)|2 (data link)|
|Comportamento|Replica sempre su tutte le porte|Forwarding selettivo basato su MAC|
|Domini di collisione|Uno solo|Uno per ogni porta|
|FDB|No|Sì|

### Processo di learning in sintesi

```
1. Arriva pacchetto su porta X con MAC sorgente S e MAC destinatario D
2. Aggiorna FDB: S → porta X (o azzera il timer se già presente)
3. Cerca D nel FDB:
   - Trovato → forwarding: invia solo sulla porta corrispondente
   - Non trovato → flooding: invia su tutte le porte tranne X
```

### Termini importanti

|Termine|Definizione|
|---|---|
|**FDB** (Filtering Database)|Tabella del bridge che mappa MAC address → porta|
|**Ageing time**|Tempo dopo cui una riga del FDB scade se non viene rinfrescata|
|**Enslaving**|Operazione di connessione di un'interfaccia al bridge software|
|**Store and forward**|Il bridge salva il pacchetto, poi decide dove inviarlo|
|**Flooding**|Invio del pacchetto su tutte le porte tranne quella di arrivo|
|**Dominio di collisione**|Segmento di rete in cui un pacchetto può causare collisioni|
|**Wireshark / Sniffer**|Tool che cattura tutti i pacchetti che raggiungono un'interfaccia|
|**Scapy**|Libreria Python per creare e inviare pacchetti di rete a basso livello|

---

## 10. Domande d'esame con risposta

**D: Che differenza c'è tra `b1` e `mainbridge` nel laboratorio?** `b1` è il nome del container assegnato da Kathará per identificare il device nella topologia. `mainbridge` è il nome dell'oggetto software bridge creato all'interno del container Linux con il comando `ip link add`. Kathará non conosce `mainbridge`.

---

**D: Perché si disabilita IPv6 nel lab.conf?** IPv6 attivo causa l'auto-assegnazione di indirizzi e l'immediato scambio di pacchetti NDP (Neighbor Discovery) tra le macchine. Questi pacchetti farebbero partire il processo di learning del bridge automaticamente, rendendo impossibile osservarlo passo passo dal FDB vuoto.

---

**D: Cosa contiene il FDB subito dopo l'avvio del bridge e prima di qualsiasi comunicazione?** Contiene solo i MAC address delle interfacce locali del bridge stesso (is local = yes, ageing timer = 0.00). Il FDB non contiene alcun MAC esterno.

---

**D: Perché nell'output di `brctl showmacs` i MAC compaiono duplicati?** È un artificio del bridge software Linux in ambiente virtualizzato Kathará. Durante lo enslaving, il software registra l'interfaccia due volte (una come interfaccia del container, una come interfaccia gestita dal bridge). Non ha effetti funzionali.

---

**D: Il Bridge fa flooding quando riceve un pacchetto la cui destinazione è nel FDB?** No. Se il MAC destinatario è presente nel FDB, il bridge fa **forwarding mirato**: invia il pacchetto solo sulla porta associata a quel MAC. Il flooding avviene solo quando il destinatario è sconosciuto.

---

**D: Cosa impara il bridge da un pacchetto ricevuto?** Il bridge impara **solo la posizione del MAC sorgente** del pacchetto, non del destinatario. Usa il MAC destinatario solo per decidere su quale porta inviare il pacchetto.

---

**D: Cosa succede all'ageing timer di una riga nel FDB quando arriva un nuovo pacchetto dallo stesso mittente?** Il timer viene **azzerato/resettato**. Finché arrivano pacchetti regolarmente da un dispositivo, la sua riga non scade mai.

---

**D: Quali interfacce vedo nell'interfaccia web di Wireshark appena avviato il lab?** Vedo `eth0` (l'interfaccia bridged che comunica col PC host) e alcune interfacce di default del container. **Non** vedo interfacce sui domini di collisione del lab finché non le aggiungo manualmente con `kathara lconfig -n wireshark --add <CD>`.

---

**D: Come si aggiunge la macchina Wireshark al dominio di collisione B?**

```bash
kathara lconfig -n wireshark --add B
```

Poi nel browser, ricaricare Wireshark con F5 o _Capture → Refresh Interfaces_. Comparirà una nuova interfaccia `eth1` (poi `eth2`, ecc. nell'ordine).

---

**D: Perché nel terzo evento (pc2 → pc1, Wireshark su C) Wireshark non cattura nulla?** Perché il bridge ha già imparato che pc1 si trova sulla porta 1 (dominio A). Quando riceve il pacchetto da pc2 destinato a pc1, lo inoltra **solo** sul dominio A. Il dominio C non viene coinvolto, quindi Wireshark attestato su C non vede il pacchetto.

---

**D: Quante porte può avere un Linux bridge?** Un bridge Linux ha un limite hardcoded nel kernel di **1024 porte**.

---

**D: Come si indica il MAC address di un'interfaccia in lab.conf?** Con la sintassi: `device[N]="NomeDominio/MAC"`. Esempio: `pc1[0]="A/00:00:00:00:00:01"` connette l'interfaccia 0 di pc1 al dominio di collisione A con MAC `00:00:00:00:00:01`.

---

_Documento generato da registrazione della lezione + slide del 19 febbraio 2026_