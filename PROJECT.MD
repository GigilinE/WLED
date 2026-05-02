# ConnectYourLife — Documentazione Tecnica Parete TV

Guida tecnica completa: hardware, chip WS2814F, problemi incontrati, soluzioni adottate, architettura finale.

---

## Hardware dell'installazione

| Componente | Dettaglio |
|---|---|
| Strip | COXO S5000-784-DRGBXK-V24-W20-P10, COB RGBW 3000K 24V |
| IC sulla strip | **WS2814F** (World-Semi, FSOP-8, marcatura "40041") |
| Pixel indirizzabili | 14 px/m × 5 m = **70 pixel** |
| LED fisici per pixel | 56 LED COB pilotati in parallelo |
| Cut unit | 71,4 mm (= 1 pixel) |
| Potenza | 20 W/m — 100 W totali per 5 m |
| Tensione strip | 24 V DC |
| Board controller | ESP32-WROOM-32 (`esp32dev`) |
| GPIO data | GPIO13, post level shifter **74AHCT125** |
| Alimentazione controller | 5 V separata (USB o PSU dedicata) |
| IP fisso | 192.168.1.74 — hostname `parete-tv.local` |
| Rete WiFi primaria | Arya |
| Firmware | ESPHome 2026.4.x |

### Strip COXO: specifiche prodotto

| Parametro | Valore |
|---|---|
| Tipo LED | COB (Chip On Board) |
| Densità | 784 LED/m |
| CCT | 3000 K (warm white) |
| CRI | Ra > 90 |
| Flusso luminoso | 1200–1300 lm/m |
| Rame PCB | 2 oz (caduta tensione ridotta) |
| Adesivo | 3M 9495LE 300MP |
| IP | IP20 (solo interno) |
| `num_leds` da configurare | **70** (14 px/m × 5 m) |

---

## Chip WS2814F — perché è diverso

Il WS2814F è la variante FSOP-8 della famiglia WS2814A di World-Semi. **Non va confuso con il WS2814 base** (SOP12, byte order RGBW). Il WS2814F ha byte order **WRGB** (W trasmesso per primo).

### Famiglia WS2814

| Variante | Package | Byte order |
|---|---|---|
| WS2814 | SOP12 | RGBW |
| WS2814A | SOP8/FSOP-8 | **WRGB** |
| WS2814B | SOP12/14 | WRGB |
| **WS2814F** | **FSOP-8** | **WRGB** |

### Parametri elettrici critici

| Parametro | Valore | Conseguenza |
|---|---|---|
| VDD | 4.5–5.5 V | Chip alimentato a 5V |
| **VIH min** | **0.7 × VDD ≈ 3.5 V** | **ESP32 a 3.3V è SOTTO SOGLIA** |
| VIL max | 1.5 V | |
| Timing | T0H 220–380 ns, T1H 580 ns–1 µs | Identico a SK6812 / WS2812B |
| Reset | ≥ 280 µs | |
| Data rate | 800 kbps | |

### Cablaggio COXO non standard

Il produttore COXO cabla i pin del WS2814F ai canali fisici in modo non conforme al datasheet:

| Byte trasmesso | Canale chip | Canale fisico COXO |
|---|---|---|
| 1° (W) | White | Bianco ✔ |
| 2° (R) | Red | B fisico |
| 3° (G) | Green | R fisico |
| 4° (B) | Blue | G fisico |

**In ESPHome con NeoPixelBus**: si usa `type: WRGB` — questo corrisponde al byte order fisico reale della strip COXO (W primo, poi i byte RGB nel loro ordine nativo). Verificato empiricamente.

### Regola fondamentale WS2814F

**Mai usare RGB e W contemporaneamente.** Il chip produce colori "sporchi" quando entrambi i gruppi sono attivi. Schema corretto:
- Bianco puro → solo W=255, RGB=0
- Colori → solo RGB, W=0

---

## Problemi incontrati e soluzioni

### Problema 1 — Colori sbagliati con byte order errato

**Sintomo:** Premendo "Rosso" si accendeva un altro colore. Il bianco W non appariva.

**Causa:** Inizialmente configurato `type: GRBW` (default SK6812). Il WS2814F vuole `WRGB` e il cablaggio COXO inverte ulteriormente i canali RGB.

**Soluzione:** `type: WRGB` in NeoPixelBus. In ESPHome questo istruisce la libreria a inviare i byte nell'ordine W, R, G, B — che corrisponde esattamente al cablaggio fisico della strip COXO.

**Nota WLED (soluzione precedente, abbandonata):** `SK6812 RGBW` + Color Order `BRG` + Swap W&G = order 34. Funzionava ma era un workaround.

---

### Problema 2 — Glitch random e flickering

**Sintomo:** Pixel che si accendono/spengono casualmente, soprattutto su animazioni veloci.

**Causa:** Fisica, non software. ESP32 emette 3.3V su GPIO. WS2814F richiede VIH min = 3.5V. Circa 200mV sotto soglia. Il chip legge bit "al limite", con errori casuali legati a temperatura e rumore.

**Soluzione hardware corretta:** Level shifter 74AHCT125 su GPIO13 (mai TXS0108E — troppo lento a 800 kbps).

**Driver scelto:** NeoPixelBus con `method: type: esp32_i2s, bus: 1` (DMA puro, immune a jitter WiFi). Il buffer DMA genera i segnali in hardware senza preemption da WiFi/BT.

---

### Problema 3 — WLED instabile con animazioni

**Sintomo:** Wipe che si ripeteva in loop, glitch all'interruzione, `repeat:1` eseguiva 2 volte, `dur:0` non era infinito (~10s).

**Causa:** Bug strutturali in WLED su animazioni one-shot con WS2814:
- Bug `repeat:1` in `playlist.cpp` (valore incrementato internamente di 1)
- `dur:0` applicava internamente un minimo di ~10s
- Effetto Wipe progettato per loop continuo, non one-shot

**Soluzione temporanea WLED:** Playlist `[Wipe → Solido]` con `dur: 4294967` (~119 ore) per il preset solido.

**Soluzione definitiva:** Migrazione completa a ESPHome.

---

### Problema 4 — ESPHome non compilava su Windows

**Sintomo:** `esphome compile` falliva con errori su `cmake`, `ninja`, `xtensa-esp32-elf` non trovati.

**Causa:** Toolchain ESP-IDF su Windows via PlatformIO aveva pacchetti stub vuoti per `tool-cmake` e `tool-ninja`.

**Soluzione:** Aggiornamento di PlatformIO e delle sue dipendenze ha risolto il problema. La toolchain ora funziona correttamente su Windows con ESPHome 2026.x.

---

### Problema 5 — Wipe OFF non partiva dopo 20 minuti di accensione

**Sintomo:** Premendo Toggle OFF dopo lungo idle, c'era una pausa percettibile prima che l'animazione iniziasse.

**Causa:** Il primo frame dell'effetto `addressable_lambda` con `initial_run = true` disegnava lo stato corrente e poi faceva `return`, aspettando il frame successivo (30ms) per iniziare a spegnere pixel.

**Soluzione:** Rimosso il `return` nel ramo `initial_run` per il wipe OFF — l'effetto disegna lo stato corrente **e** spegne il primo pixel nello stesso frame, senza aspettare.

```cpp
if (initial_run) {
  // disegna stato corrente...
  if (dir == 1) return;  // wipe ON: aspetta frame successivo (ok)
  // wipe OFF: cade attraverso e inizia subito
}
```

---

### Problema 6 — Quick toggle: premere ON→OFF subito perdeva l'animazione

**Sintomo:** Se si premeva Toggle OFF mentre il Wipe ON era ancora in corso, lo spegnimento non avveniva.

**Causa:** Il codice originale resettava `wipe_pos = 0` prima di impostare `wipe_direction = -1`. Il wipe OFF con `pos=0` e `initial_run=false` chiamava immediatamente `turn_off()` senza animare.

**Soluzione:** Eliminato il reset di `wipe_pos` nel percorso di spegnimento. Il wipe OFF parte dalla posizione corrente e retrocede — se il wipe ON era a metà, lo spegnimento inverte da lì.

---

### Problema 7 — Wipe OFF portava la luminosità a 100% prima di animare

**Sintomo:** Premendo Toggle OFF con dimmer al 40%, la strip saltava brevemente al 100% poi avviava il wipe.

**Causa:** Il percorso `on_turn_off` chiamava `light.turn_on(brightness: 100%, effect: "Wipe White")`.

**Soluzione:** Rimosso `brightness: 100%` dal percorso di spegnimento. Il wipe OFF usa qualsiasi luminosità era attiva. La luminosità al 100% è rimasta solo sul percorso di accensione (wipe ON).

---

### Problema 8 — Il bianco diventava "freddo" (RGB invece di W)

**Sintomo:** Dopo aver completato il wipe ON, o premendo il pulsante "Bianco", la strip mostrava un bianco freddo RGB invece del bianco caldo del canale W.

**Causa (radice):** `light.turn_on(white: 100%)` via YAML API popola `current_values` con un mix RGB+W determinato da ESPHome internamente. L'effetto "Solido" leggeva `current_values.get_white()` ottenendo valori errati. Il risultato era R≈G≈B>0 invece di W=255.

**Soluzione:** Aggiunta di effetti "Statico X" che scrivono **direttamente** `Color(r,g,b,w)` nel buffer pixel, bypassando completamente `current_values`:

```cpp
// Statico Bianco — puro canale W, nessun mixing RGB
for (int i = 0; i < it.size(); i++) it[i] = Color(0, 0, 0, 255);
```

Il wipe ON ora termina con `call.set_effect("Statico Bianco")` invece di impostare colori via API YAML.

I pulsanti Bianco/Rosso/Verde/Blu usano `effect: "Statico X"` senza passare parametri RGBW a YAML.

---

### Problema 9 — Wipe OFF cambiava colore

**Sintomo:** Il wipe OFF mostrava bianco invece del colore correntemente visualizzato.

**Causa:** L'effetto leggeva il colore da `current_values` che, dopo YAML `white: 100%`, conteneva valori RGB mescolati.

**Soluzione:** Lettura del colore direttamente dal **buffer pixel** (`it[0].get()`) che contiene sempre il colore reale scritto nell'ultimo frame:

```cpp
auto c = it[0].get();
on_color = Color(c.r, c.g, c.b, c.w);
```

---

## Architettura finale adottata

### Stack tecnologico

```
[PSU 24V Meanwell] ──► [Strip COXO WS2814F, 70px, 5m]
                              ▲ DATA 5V
                              │
                    [74AHCT125 level shifter]
                              │ DATA 3.3V
                              │
                   [ESP32-WROOM-32, GPIO13]
                              │
                       ESPHome 2026.4.x
                       NeoPixelBus I2S DMA
                              │
                   WiFi "Arya" → 192.168.1.74
                              │
                   Web UI porta 80 (web_server v3)
                   API ESPHome (porta 6053)
                   Home Assistant (via API)
```

### Driver LED scelto

```yaml
platform: neopixelbus
type: WRGB        # byte order chip: W, R, G, B
variant: SK6812   # timing NRZ 800kHz compatibile
method:
  type: esp32_i2s
  bus: 1          # DMA puro — immune a jitter WiFi
```

### Stato machine animazione — globali

| Variabile | Tipo | Scopo |
|---|---|---|
| `wipe_pos` | int | Posizione corrente del wipe (pixel 0–70) |
| `wipe_direction` | int | 1 = accensione A→B, -1 = spegnimento B→A |
| `skip_anim` | bool | Bypassa `on_turn_on` durante riaccensione interna |
| `wipe_off_running` | bool | True mentre il wipe OFF è in corso |
| `booted` | bool | True dopo boot — evita lo slider luminosità al boot |

### Effetti disponibili

| Effetto | Tipo | Canale usato | Note |
|---|---|---|---|
| **Statico Bianco** | lambda | W puro | `Color(0,0,0,255)` hardcoded — no mixing |
| **Statico Rosso** | lambda | R puro | `Color(255,0,0,0)` hardcoded |
| **Statico Verde** | lambda | G puro | `Color(0,255,0,0)` hardcoded |
| **Statico Blu** | lambda | B puro | `Color(0,0,255,0)` hardcoded |
| **Solido** | lambda | RGBW | Legge `current_values` — per color picker live |
| **Respiro Bianco** | lambda | W | Sine wave su canale W |
| **Fade Colori** | lambda | RGB | Tre sine wave sfasate di 120° |
| **Tramonto** | lambda | RGB+W | Toni arancio-rosso caldi |
| **Oceano** | lambda | RGB | Onda blu-verde per pixel |
| **Disco** | lambda | RGB | Colori random rapidi |
| **Rainbow** | built-in | RGB | Arcobaleno rotante |
| **Wipe White** | lambda | W/colore | **Uso interno — toggle animato** |

> **Statico X vs Solido**: gli effetti Statico scrivono direttamente nel buffer pixel e non soffrono del problema `current_values`. Usarli sempre per i preset colore. Usare Solido solo per il color picker interattivo.

### Logica toggle — `animated_toggle` script

```
Toggle ON (luce spenta):
  → light.turn_on(brightness:100%) → on_turn_on → wipe_pos=0, dir=1, effect="Wipe White"
  → wipe avanza pixel per pixel (W=255)
  → al pixel 70: set_effect("Statico Bianco") — rimane accesa

Toggle OFF (luce accesa):
  → wipe_off_running=true, dir=-1, effect="Wipe White" (skip on_turn_on)
  → wipe retrocede dal pixel corrente
  → al pixel 0: turn_off() → on_turn_off(else) → wipe_off_running=false

Toggle OFF durante Wipe ON:
  → wipe_direction=-1 — il wipe inverte dalla posizione corrente
  → nessun reset di wipe_pos

Toggle ON durante Wipe OFF:
  → script mode:restart — lo script riparte, avvia wipe ON
```

### Regole critiche per modifiche future

1. **Mai** passare parametri RGBW via YAML a un effetto Statico (li sovrascrive `current_values`)
2. **Non resettare `wipe_pos`** nel percorso di spegnimento — il wipe OFF parte dalla posizione corrente
3. **`skip_anim = true`** prima di qualunque `light.turn_on` interno che non deve triggherare `on_turn_on`
4. **Leggere colore per wipe OFF** sempre da `it[0].get()`, mai da `current_values`
5. **Wipe White non va rimosso** dagli effetti — è usato internamente dal toggle; confondere l'utente è accettabile

---

## Parametri di compilazione e upload

```bash
# OTA (dispositivo in rete)
esphome upload parete-tv.yaml --device 192.168.1.74

# Primo flash via USB
esphome upload parete-tv.yaml --device COM5

# Solo compilazione (senza upload)
esphome compile parete-tv.yaml
```

Il firmware compilato si trova in:
`.esphome/build/parete-tv/.pioenvs/parete-tv/firmware.bin` (OTA)
`.esphome/build/parete-tv/.pioenvs/parete-tv/firmware.factory.bin` (primo flash)

---

## Riferimenti tecnici

### Datasheet

- WS2814 IC Datasheet V1.4: `WS2814F 40041.pdf` (in repo)
- Strip COXO S5000: `S5000-784-DRGBXK-V24-NWP-W20-P10-A2.pdf` (in repo)

### ESPHome

- NeoPixelBus component: esphome.io/components/light/neopixelbus
- web_server v3: esphome.io/components/web_server
- addressable_lambda: esphome.io/components/light/neopixelbus#addressable-light-effects
- Bug default_transition_length #6621: github.com/esphome/esphome/issues/6621

### NeoPixelBus (Makuna)

- github.com/Makuna/NeoPixelBus
- WRGB support (v2.8.0+): github.com/Makuna/NeoPixelBus/releases

### Community WS2814

- WLED issue WS2814 support: github.com/Aircoookie/WLED/issues/3655
- WS2814 color order thread: github.com/Aircoookie/WLED/issues/2646
- WS2814 24V 3.3V flickering: wled.discourse.group/t/ws2814-24v-flickering-gliching-when-connected-to-3-3v-no-change-with-logic-level-shifter/10854

---

## Note installazione

- **Alimentatore strip**: PSU 24V dedicata, Meanwell LRS-150-24 (150W) consigliato per singola strip 5m (100W + 50% margine)
- **Level shifter**: 74AHCT125 obbligatorio. Non usare TXS0108E (troppo lento a 800kbps). Non usare 4050 (non veloce abbastanza)
- **Cavo dati**: ≤ 1m dall'ESP al primo pixel
- **Ground comune**: PSU 5V ESP e PSU 24V strip devono condividere GND
- **Power injection**: consigliata all'estremità opposta per bianchi saturi costanti su tutta la lunghezza

---

*ConnectYourLife — documentazione interna 2026*
