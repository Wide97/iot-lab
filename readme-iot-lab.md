# IoT Lab con Docker, Mosquitto e Node-RED

Questo progetto crea un laboratorio IoT locale con:
- broker MQTT `Mosquitto`
- orchestrazione flow e dashboard con `Node-RED`

## Struttura progetto

```text
iot-lab/
├── docker-compose.yml
├── flows.json
├── mosquitto/
│   └── mosquitto.conf
└── readme-iot-lab.md
```

## Prerequisiti

- Docker Desktop installato
- In Windows + WSL2: integrazione WSL attiva in Docker Desktop (`Settings -> Resources -> WSL Integration`)
- VS Code con estensione `Remote - WSL`

## Avvio del laboratorio

Dalla root del progetto:

```bash
docker compose up -d
```

Per verificare:

```bash
docker compose ps
docker compose logs -f mosquitto
docker compose logs -f nodered
```

## Accesso ai servizi

- Node-RED editor: `http://localhost:1880`
- MQTT broker: `localhost:1883`

## Installazione dashboard Node-RED

Il flow usa nodi dashboard (`ui_gauge`, `ui_text`), quindi installa `node-red-dashboard` nel container Node-RED:

```bash
docker exec -it iot-lab-nodered bash -lc "cd /data && npm install node-red-dashboard"
docker restart iot-lab-nodered
```

Alternativa da UI Node-RED:
- Menu `Manage palette` -> `Install` -> cerca `node-red-dashboard`.

## Importare il flow `flows.json`

1. Apri `http://localhost:1880`
2. Menu in alto a destra -> `Import`
3. Seleziona il contenuto di `flows.json`
4. Conferma import e clicca `Deploy`

Dashboard disponibile su:
- `http://localhost:1880/ui`

## Cosa fa il flow (nodo per nodo)

### Pipeline principale

1. `Tick sensore ogni 5s` (`inject`)
- Emette un trigger ogni 5 secondi.

2. `Genera valori sensori random` (`function`)
- Simula dati elettrici:
  - tensione (`voltage_v`)
  - corrente (`current_a`)
  - power factor (`power_factor`)

3. `Calcola potenza` (`function`)
- Calcola:
  - potenza apparente (`VA`)
  - potenza attiva (`W`)
  - potenza reattiva (`var`)

4. `Pubblica metriche MQTT` (`mqtt out`)
- Pubblica JSON su topic `iot/lab/power/metrics`.

5. `Verifica soglia potenza` (`switch`)
- Se `active_power_w >= 800`: ramo allarme
- Se `active_power_w < 800`: ramo normale

6. `Stato: allarme` / `Stato: normale` (`change`)
- Imposta messaggio stato testuale e topic `iot/lab/power/status`.

7. `Pubblica stato MQTT` (`mqtt out`)
- Pubblica lo stato su MQTT.

### Dashboard

- `Estrai active_power_w` -> `Gauge Potenza Attiva` (`ui_gauge`)
- `Estrai voltage_v` -> `Gauge Tensione` (`ui_gauge`)
- `Indicatore soglia` (`ui_text`) mostra `OK`/`ATTENZIONE`
- `Dettaglio misure` (`ui_text`) mostra riepilogo misura

## Configurazione Mosquitto

In `mosquitto/mosquitto.conf`:
- `listener 1883`: porta MQTT standard
- `allow_anonymous true`: accesso senza credenziali (solo laboratorio)
- log su volume persistente (`/mosquitto/log/mosquitto.log`)
- persistenza dati in `/mosquitto/data`

## Best practice (WSL/VS Code/Git/MQTT)

### WSL + VS Code

- Lavora su file dentro filesystem Linux (es. `/home/...`) per performance migliori.
- Apri la cartella con `Remote - WSL` (non da path Windows `C:\...`).
- Evita mount non necessari in tempo reale tra Windows e Linux.

### Git e configurazioni

- Versiona file di infrastruttura: `docker-compose.yml`, `mosquitto.conf`, `flows.json`.
- Evita di committare dati runtime/log temporanei.
- Crea commit piccoli e descrittivi (es. `feat: add nodered sample flow`).

### MQTT topic naming

- Usa namespace stabili: `<org>/<site>/<device>/<metric>`
- Esempio: `iot/lab/power/metrics`, `iot/lab/power/status`
- Mantieni convenzione coerente (`snake_case` o `kebab-case`, non mista).
- Se utile, separa topic telemetria da comandi:
  - `.../telemetry/...`
  - `.../cmd/...`

## Snippet utile per Dev Container (opzionale)

Se vuoi lavorare sempre con tool coerenti in VS Code, puoi aggiungere:

```json
{
  "name": "iot-lab-dev",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "nodered",
  "workspaceFolder": "/workspace",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-azuretools.vscode-docker",
        "ms-vscode-remote.remote-wsl"
      ]
    }
  }
}
```

Nota: è uno snippet di base, da adattare alla tua struttura `.devcontainer/`.
