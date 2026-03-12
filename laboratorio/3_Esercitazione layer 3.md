## Routing

### `ip route` — Gestione tabella di routing

```
ip route [show]                          # mostra la tabella di routing
ip route add NETWORK/MASK via GATEWAY    # aggiunge una route verso una rete
ip route add NETWORK/MASK dev DEV        # aggiunge una route su un'interfaccia diretta
ip route add default via GATEWAY         # aggiunge il default gateway
ip route del NETWORK/MASK [via GATEWAY]  # rimuove una route
```

**Esempi:**

```bash
ip route
ip route add 10.0.0.0/8 via 192.168.1.1
ip route add 10.0.0.0/8 dev eth1
ip route add default via 192.168.1.1
ip route del 10.0.0.0/8 via 192.168.1.1
```

> **Nota:** `ip route add default` e `ip route add 0.0.0.0/0` sono equivalenti.
