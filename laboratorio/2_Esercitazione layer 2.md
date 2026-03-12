### `ip link` — Stato interfacce (layer 2)

```
ip link show [DEV]           # mostra stato interfacce (tutte, o solo DEV)
ip link set DEV up           # attiva l'interfaccia DEV
ip link set DEV down         # disattiva l'interfaccia DEV
```

**Esempi:**

```bash
ip link show
ip link show eth0
ip link set eth0 up
ip link set eth0 down
```
