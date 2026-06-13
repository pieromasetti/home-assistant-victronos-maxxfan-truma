# 🚐 GENIUS_VAN — Home Assistant per Campervan

Sistema completo di monitoraggio e controllo per campervan basato su **Home Assistant** in esecuzione su **Arduino UNO Q**, con integrazione **Victron**, sensori ambientali, meteo dinamico via GPS e accesso remoto sicuro.

> ⚠️ **Nota sui dati sensibili**: questo documento usa segnaposto (es. `LA_TUA_PASSWORD`) al posto di password, chiavi e indirizzi reali. Sostituiscili con i tuoi valori. **Non condividere mai pubblicamente le tue credenziali reali.**

---

## 📑 Indice

1. [Panoramica del sistema](#panoramica)
2. [Hardware](#hardware)
3. [Installazione di Home Assistant](#installazione-ha)
4. [Accesso remoto con Tailscale](#tailscale)
5. [Integrazione Victron](#victron)
6. [Sensori temperatura RuuviTag](#ruuvitag)
7. [Sensori template (decimali e autonomia)](#template)
8. [Meteo dinamico via GPS](#meteo)
9. [Allerte meteo](#allerte)
10. [Card personalizzate (HACS)](#hacs)
11. [Dashboard completa](#dashboard)
12. [Progetti ESPHome futuri](#esphome)

---

<a name="panoramica"></a>
## 1. Panoramica del sistema

| Componente | Tecnologia |
|---|---|
| **Cervello** | Home Assistant (Docker) su Arduino UNO Q |
| **Energia** | Impianto Victron (via Venus OS su Raspberry Pi) |
| **Comunicazione Victron** | MQTT |
| **Sensori ambientali** | RuuviTag (BLE) |
| **Posizione** | GPS NMEA-0183 |
| **Meteo** | Open-Meteo (dinamico, segue il GPS) |
| **Allerte** | MeteoAlarm (Europa) |
| **Accesso remoto** | Tailscale (VPN mesh) |
| **Reti** | WiFi van (Netgear Nighthawk 5G) + Starlink |

**Filosofia**: tutto gira **localmente** nel veicolo. Nessuna dipendenza dal cloud per le funzioni core. Il sistema funziona anche offline quando si è nel raggio del WiFi del van.

---

<a name="hardware"></a>
## 2. Hardware

- **Arduino UNO Q** (4GB RAM, 32GB eMMC) — host di Home Assistant
- **Raspberry Pi** con **Venus OS** — gateway impianto Victron
- **Impianto Victron**:
  - SmartShunt 300A (monitor batterie LiFePO4)
  - SmartSolar MPPT 75/15 (pannelli solari)
  - MultiPlus 12/2000/80-32 (inverter/caricabatterie)
  - Orion XS 12/12-50A (DC/DC charger)
- **RuuviTag** x4 (temperatura/umidità: interno, esterno, frigo, freezer)
- **GPS NMEA-0183** (USB)
- **Netgear Nighthawk M5** (router 5G) + **Starlink**
- **Tablet Realme** (Android) come display dedicato

---

<a name="installazione-ha"></a>
## 3. Installazione di Home Assistant

L'Arduino UNO Q esegue **Debian Linux** con Docker preinstallato. Home Assistant gira come container.

### Verifica Docker

```bash
docker --version
```

### Installazione Home Assistant Container

```bash
docker run -d \
  --name homeassistant \
  --privileged \
  --restart=unless-stopped \
  -e TZ=Europe/Rome \
  -v /home/arduino/homeassistant:/config \
  --network=host \
  ghcr.io/home-assistant/home-assistant:stable
```

### Accesso

Apri il browser su `http://IP_ARDUINO:8123` e completa la configurazione iniziale.

> 💡 **Spazio disco**: l'UNO Q ha una partizione root limitata (~10GB). Se lo spazio si esaurisce, libera le immagini Docker inutilizzate con:
> ```bash
> docker image prune -a -f
> ```

---

<a name="tailscale"></a>
## 4. Accesso remoto con Tailscale

Tailscale crea una VPN mesh sicura tra tutti i dispositivi, permettendo l'accesso a HA da qualsiasi luogo (5G/Starlink) senza aprire porte.

### Installazione su Arduino UNO Q

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Segui il link di autenticazione per collegare il dispositivo al tuo account Tailscale.

### Trova l'IP Tailscale

```bash
tailscale ip
```

### Accesso

- Installa l'app **Tailscale** su telefono/tablet (stesso account)
- Accedi a HA via `http://IP_TAILSCALE:8123` da qualsiasi rete

> 📝 **Nota**: l'app Home Assistant funziona sia in locale (WiFi del van) sia da remoto (via Tailscale). Imposta la dashboard del van come predefinita e nascondi la barra laterale per un'esperienza pulita.

---

<a name="victron"></a>
## 5. Integrazione Victron

L'impianto Victron è gestito da **Venus OS** (su Raspberry Pi), che espone tutti i dati via **MQTT**.

### Passo 1 — Abilita MQTT sul Venus OS

Nell'interfaccia Venus OS: **Settings → Services → MQTT on LAN (Plaintext)** → ON

### Passo 2 — Aggiungi l'integrazione MQTT in HA

**Impostazioni → Dispositivi e servizi → Aggiungi integrazione → MQTT**

- **Broker**: `IP_VENUS_OS`
- **Porta**: `1883`
- Utente/password: vuoti (se non configurati)

### Passo 3 — Installa l'integrazione Victron MQTT (via HACS)

In HACS cerca **"Victron MQTT Integration"** e installala. Dopo il riavvio, HA rileva automaticamente il dispositivo GX e tutte le entità Victron (19 dispositivi, ~249 entità nel setup di esempio).

> ⚠️ **Nomi entità**: l'integrazione può creare entità con suffisso `_2` (es. `sensor.smartshunt_300a_id_290_charge_2`). Verifica i nomi esatti nel registro entità prima di costruire la dashboard. Comando utile:
> ```bash
> docker exec homeassistant grep -o '"entity_id":"sensor.smartshunt[^"]*"' /config/.storage/core.entity_registry
> ```

---

<a name="ruuvitag"></a>
## 6. Sensori temperatura RuuviTag

I RuuviTag sono sensori BLE che HA rileva automaticamente tramite l'integrazione **RuuviTag BLE** (richiede un adattatore Bluetooth o un Bluetooth proxy ESPHome).

Nel setup di esempio i RuuviTag sono rinominati: `Interno VAN`, `Temperatura Esterna`, `Frigo`, `Freezer`.

---

<a name="template"></a>
## 7. Sensori template

### Autonomia batteria (da secondi a giorni/ore)

Il SmartShunt fornisce l'autonomia in secondi. Questo template la converte in formato leggibile e mostra `N/A` quando la batteria è in carica o quasi piena.

```yaml
template:
  - sensor:
      - name: 'Autonomia Batteria'
        unique_id: autonomia_batteria
        state: >
          {% set sec = states('sensor.smartshunt_300a_id_290_time_to_go_2') | int(0) %}
          {% if sec == 0 or states("sensor.smartshunt_300a_id_290_dc_bus_current_2") | float(0) >= 0 or states("sensor.smartshunt_300a_id_290_charge_2") | float(0) >= 99 %}
            N/A
          {% else %}
            {% set gg = (sec // 86400) %}
            {% set ore = ((sec % 86400) // 3600) %}
            {% set min = ((sec % 3600) // 60) %}
            {% if gg > 0 %}{{ gg }}g {{ ore }}h{% else %}{{ ore }}h {{ min }}m{% endif %}
          {% endif %}
        icon: mdi:clock-outline
```

### Arrotondamento temperature e AC

```yaml
  - sensor:
      - name: Temperatura Interno
        state: "{{ states('sensor.interno_van_temperature') | float(0) | round(1) }}"
        unit_of_measurement: "°C"
      - name: Temperatura Esterno
        state: "{{ states('sensor.temperatura_esterna_temperature') | float(0) | round(1) }}"
        unit_of_measurement: "°C"
      - name: Temperatura Frigo
        state: "{{ states('sensor.frigo_temperature') | float(0) | round(1) }}"
        unit_of_measurement: "°C"
      - name: Temperatura Freezer
        state: "{{ states('sensor.freezer_temperature') | float(0) | round(1) }}"
        unit_of_measurement: "°C"
      - name: AC Tensione
        state: "{{ states('sensor.multiplus_12_2000_80_32_id_288_input_voltage_l1_2') | float(0) | round(1) }}"
        unit_of_measurement: "V"
```

> Aggiungi questi blocchi in `configuration.yaml` e riavvia HA.

---

<a name="meteo"></a>
## 8. Meteo dinamico via GPS

Il meteo segue automaticamente la posizione del van usando **Open-Meteo** e una **zona mobile** aggiornata dal GPS.

### Passo 1 — Crea una zona (es. "GeniusVan")

**Impostazioni → Aree, etichette e zone → Zone → Aggiungi zona**

### Passo 2 — Installa Open-Meteo

**Impostazioni → Dispositivi e servizi → Aggiungi integrazione → Open-Meteo**, collegandola alla zona creata.

### Passo 3 — Abilita python_script

Aggiungi a `configuration.yaml`:

```yaml
python_script:
```

### Passo 4 — Script che sposta la zona seguendo il GPS

File `python_scripts/update_zone.py`:

```python
tracker = hass.states.get('device_tracker.nmea_0183_gps_acm0_location_2')
if tracker is not None:
    lat = tracker.attributes.get('latitude')
    lon = tracker.attributes.get('longitude')
    if lat is not None and lon is not None:
        hass.states.set('zone.geniusvan', 'zoning', {
            'latitude': lat,
            'longitude': lon,
            'radius': 1000,
            'friendly_name': 'Genius Van',
            'icon': 'mdi:rv-truck'
        })
```

### Passo 5 — Automazione (aggiornamento periodico)

```yaml
alias: Aggiorna meteo Van da GPS
description: Sposta la zona seguendo il GPS del van
triggers:
  - trigger: time_pattern
    minutes: "/30"
conditions:
  - condition: template
    value_template: >
      {{ state_attr('device_tracker.nmea_0183_gps_acm0_location_2', 'latitude') is not none }}
actions:
  - action: python_script.update_zone
mode: single
```

Risultato: l'entità `weather.geniusvan` mostra il meteo aggiornato sulla posizione corrente del van.

---

<a name="allerte"></a>
## 9. Allerte meteo

**MeteoAlarm** fornisce allerte ufficiali per l'Europa. Aggiungi a `configuration.yaml`:

```yaml
binary_sensor:
  - platform: meteoalarm
    country: "italy"
    province: "Emilia Romagna"
    language: "it-IT"
```

> ⚠️ MeteoAlarm richiede paese/provincia **fissi**. Per un van che viaggia, la regione va aggiornata manualmente, oppure ci si affida al solo meteo dinamico di Open-Meteo.

La dashboard mostra una card di allerta **solo quando attiva** (vedi sezione Dashboard).

---

<a name="hacs"></a>
## 10. Card personalizzate (HACS)

### Installa HACS

```bash
docker exec homeassistant bash -c "wget -O - https://get.hacs.xyz | bash -"
docker restart homeassistant
```

Poi **Impostazioni → Dispositivi e servizi → Aggiungi integrazione → HACS**.

### Card usate in questo progetto

| Card | Repository | Uso |
|---|---|---|
| **Mushroom** | `piitaya/lovelace-mushroom` | Card moderne |
| **Mini Graph Card** | `kalkih/mini-graph-card` | Grafico batteria |
| **Bubble Card** | `Clooos/Bubble-Card` | Controlli e separatori |
| **Button Card** | `custom-cards/button-card` | Tile personalizzati |
| **card-mod** | `thomasloven/lovelace-card-mod` | Stili CSS custom |

> 💡 Le card si distribuiscono come file `.js`. Se HACS non riesce a scaricarle, si possono installare manualmente in `/config/www/community/` e registrarle in **Impostazioni → Dashboard → Risorse**.

---

<a name="dashboard"></a>
## 11. Dashboard completa

Stile "cruscotto" scuro con gauge colorati, badge, tile, meteo e mappa. Layout `sections` per adattarsi a tablet e telefono.

> Il file YAML completo della dashboard è incluso separatamente: **`dashboard-genius-van.yaml`**

Caratteristiche principali:
- **Badge** in alto: tensione, potenza, temperature, autonomia
- **Gauge** semicircolari: batteria, solare, frigo, freezer
- **Tile** raggruppati: energia, temperature, controlli
- **Grafico** carica batteria 24h
- **Meteo** dinamico con previsioni
- **Allerta meteo** condizionale (appare solo se attiva)
- **Mappa** GPS in tempo reale
- **Vista Controlli** separata: MultiPlus (modalità, PowerAssist, limite corrente), Orion XS, relè GX

---

<a name="esphome"></a>
## 12. Progetti ESPHome futuri

Espansioni pianificate con **M5Stack NanoC6** (ESP32-C6) ed ESPHome:

| Progetto | Metodo | Stato |
|---|---|---|
| **MaxxFan** | IR integrato NanoC6 | Config pronta |
| **Truma Combi 6D / iNet X** | LIN bus via TJA1020 | Config pronta (sperimentale) |
| **OBD Mercedes** | MeatPi YCAN / ESP32 OBD | Pianificato |
| **Livella van** | NanoC6 + MPU6050 (IMU) | Pianificato |

### Installazione ESPHome (container Docker)

```bash
docker run -d \
  --name esphome \
  --restart=unless-stopped \
  --network=host \
  -v /home/arduino/esphome:/config \
  -e TZ=Europe/Rome \
  ghcr.io/esphome/esphome:stable
```

Interfaccia web su `http://IP_ARDUINO:6052`.

> I file di configurazione per MaxxFan e Truma sono inclusi separatamente:
> - **`maxxfan-nanoc6.yaml`**
> - **`truma-nanoc6.yaml`**

### Note tecniche NanoC6
- Solo WiFi **2.4GHz** (non 5GHz)
- IR LED integrato (per MaxxFan)
- Connettore Grove per UART/I2C (per TJA1020 LIN, sensori)
- UART a 3.3V — il TJA1020 non richiede level shifter

---

## 🙏 Crediti e ispirazione

- **Smarty Van** (YouTube) — ispirazione dashboard e progetto MaxxFan/OBD
- **Fabian-Schmidt/esphome-truma_inetbox** — componente Truma
- **mc0110/inetbox2mqtt** — protocollo Truma + livella
- **DanStasiak/caravan-home-assistant** — riferimento livella e struttura
- Community **Home Assistant** e **ESPHome**

---

## 📄 Licenza

Condividi liberamente. Adatta alla tua configurazione. Le credenziali sono tue — non pubblicarle mai.

*Documento generato per il progetto GENIUS_VAN. 🚐☀️*
