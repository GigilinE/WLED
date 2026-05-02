# ConnectYourLife — Parete TV

Firmware ESPHome per strip COB WS2814F RGBW 24V su ESP32 — installazione **Parete TV**.

## Hardware

| Componente | Dettaglio |
|---|---|
| Board | ESP32-WROOM-32 (`esp32dev`) |
| Strip | COXO S5000-784-DRGBXK-V24-W20-P10, COB RGBW 3000K 24V |
| IC strip | WS2814F (FSOP-8, marcatura "40041") — byte order WRGB |
| Pixel | 14 px/m × 5 m = **70 pixel** |
| GPIO data | GPIO13 — post level shifter 74AHCT125 |
| Driver firmware | NeoPixelBus I2S DMA (bus 1) — immune a jitter WiFi |
| IP | 192.168.1.74 (`parete-tv.local`) |

## Quick Start

```bash
# Installa ESPHome
pip install esphome

# Copia e compila le credenziali
cp secrets.yaml.example secrets.yaml
# Modifica secrets.yaml con i tuoi valori

# Upload OTA (dispositivo già flashato)
esphome upload parete-tv.yaml --device 192.168.1.74

# Primo flash via USB (COM5)
esphome upload parete-tv.yaml --device COM5
```

## Funzionalità

- **On/off animato**: wipe A→B all'accensione (bianco caldo canale W), wipe B→A allo spegnimento nel colore corrente
- **Toggle interruzione-sicuro**: se si preme ON durante un wipe OFF (o viceversa), l'animazione inverte da dove si trova
- **Dimmer** preservato durante on/off — lo spegnimento non porta mai a 100% prima di animare
- **Color picker** live RGBW via web UI (effetto Solido)
- **Preset colore** dedicati: Bianco (W puro), Rosso, Verde, Blu — scrittura diretta sul buffer, mai mescolati
- **Effetti decorativi**: Rainbow, Respiro Bianco, Fade Colori, Tramonto, Oceano, Disco
- Web UI locale brandizzata ConnectYourLife (web_server v3, porta 80)
- Integrazione nativa Home Assistant (API ESPHome)
- Aggiornamenti OTA

## File principali

| File | Scopo |
|---|---|
| `parete-tv.yaml` | Config ESPHome — unico device attivo |
| `secrets.yaml` | Credenziali (WiFi, API, OTA) — non committare |
| `secrets.yaml.example` | Template credenziali |
| `common/base.yaml` | WiFi, API, OTA, web_server — condiviso |
| `custom/connectyourlife.css` | Stile web UI |
| `custom/connectyourlife.js` | Logica web UI |

## Documentazione tecnica

Vedi [PROJECT.MD](PROJECT.MD) per: chip WS2814F, problemi incontrati, soluzioni, architettura finale.

## Licenza

EUPL-1.2 — vedi [LICENSE](LICENSE)
