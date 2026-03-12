### Collision Domain (dominio di collisione)

Un collision domain è un segmento di rete virtuale condiviso — l'equivalente virtuale di un cavo o di uno switch a cui colleghi più dispositivi.

In Katharà, ogni collision domain è identificato da un nome (es. `A`, `LAN1`, `net_core`). Tutti i device che hanno un'interfaccia collegata allo stesso collision domain si trovano sulla stessa rete virtuale e possono comunicare tra loro a livello L2, a patto che le interfacce siano configurate con indirizzi IP compatibili.

```
kathara vstart -n pc1 --eth 0:A
kathara vstart -n pc2 --eth 0:A
# pc1 e pc2 sono ora sulla stessa rete virtuale "A"
# e possono comunicare (dopo aver configurato gli IP)
```

> **Nota:** Il nome "collision domain" viene dalla terminologia Ethernet classica, dove i dispositivi collegati allo stesso mezzo fisico condividevano il canale e potevano generare collisioni. In Katharà è semplicemente l'etichetta che identifica una rete virtuale.