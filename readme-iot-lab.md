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

4. `Pubblica dati raw MQTT` (`mqtt out`)
- Pubblica dati grezzi su topic `iot/lab/power/raw`.

5. `Pubblica metriche MQTT` (`mqtt out`)
- Pubblica JSON con valori calcolati su topic `iot/lab/power/metrics`.

6. `Verifica soglia potenza` (`switch`)
- Se `active_power_w >= 800`: ramo allarme
- Se `active_power_w < 800`: ramo normale

7. `Stato: allarme` / `Stato: normale` (`change`)
- Imposta messaggio stato testuale e topic `iot/lab/power/status`.

8. `Pubblica stato MQTT` (`mqtt out`)
- Pubblica lo stato su MQTT.

9. `Crea evento allarme` + `Pubblica allarme MQTT` (`function` + `mqtt out`)
- In caso di superamento soglia, pubblica evento su `iot/lab/power/alarm`.

### Dashboard

- `Estrai active_power_w` -> `Gauge Potenza Attiva` (`ui_gauge`)
- `Estrai voltage_v` -> `Gauge Tensione` (`ui_gauge`)
- `Indicatore soglia` (`ui_text`) mostra `OK`/`ATTENZIONE`
- `Dettaglio misure` (`ui_text`) mostra riepilogo misura
- `Indicatore raw` (`ui_text`) mostra l'ultimo pacchetto grezzo generato
- `Indicatore ultimo allarme` (`ui_text`) mostra l'ultimo evento sopra soglia

## Integrazione Tapo L530 (controllo reale)

Il file `flows.json` include anche un tab `Tapo Lab - Lampadina` con:
- poll periodico stato lampadina
- controllo ON/OFF da dashboard
- controllo brightness da dashboard
- publish stato su topic MQTT `iot/lab/tapo/l530/status`

Per usare i nodi Tapo:
1. Installa `node-red-contrib-tplink-tapo-connect-api` nel container Node-RED.
2. Importa `flows.json` e fai deploy.
3. Apri i nodi `tplink_status`, `tplink_turn_on`, `tplink_turn_off`, `tplink_brightness`.
4. Inserisci configurazione device (`searchMode: ip` e IP lampadina L530).
5. Inserisci le credenziali account Tapo richieste dai nodi.

Nota sicurezza:
- Le credenziali Tapo non vanno committate su Git.
- Tienile solo nella configurazione runtime di Node-RED (`flows_cred.json`).

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
- Esempi usati in questo lab:
  - `iot/lab/power/raw`
  - `iot/lab/power/metrics`
  - `iot/lab/power/status`
  - `iot/lab/power/alarm`
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
