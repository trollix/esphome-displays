# Dashboard — JC3248W535C pour Home Assistant

Firmware ESPHome pour le **JC3248W535C** — ESP32-S3 3.5" capacitif, 8 MB PSRAM.  
Dashboard tactile **une seule page portrait** affichant températures, énergie et charge véhicule électrique, connecté à Home Assistant.

Évolution du projet [thermometre-cyd](../thermometre-cyd/README.md) (ESP32-3248S035C), porté sur ESP32-S3 avec PSRAM pour une réactivité nettement améliorée et un affichage unifié en mode portrait natif.

![exemple d'affichage](docs/screenshot.jpg)

---

## Matériel

- **Carte** : JC3248W535C — ESP32-S3-WROOM-1 + écran 3.5" 320×480 IPS + tactile capacitif AXS15231B
- **PSRAM** : 8 MB octal intégrée dans l'ESP32-S3R8
- **Flash** : 16 MB
- **Connecteur** : USB-C
- **Alimentation** : chargeur 5V/2A via USB-C

Compter ~25-30 € sur Amazon (DIYmalls), ~15-20 € sur AliExpress.

---

## Fonctionnalités

- **1 seule page portrait 320×480** — toutes les infos d'un seul coup d'œil, sans swipe
- **Températures** : extérieur, intérieur, piscine avec couleurs contextuelles selon les seuils
- **Énergie** : production solaire (±W), consommation maison (W)
- **Tarif HP/HC** affiché dans le header en couleur (vert = HC, orange = HP)
- **Charge Zoé** : % batterie + km d'autonomie avec code couleur (vert/orange/rouge)
- **Veille automatique** : dim à 60% après 3 minutes d'inactivité (configurable via substitution)
- **Réveil au tap** : tap = retour à 100%, reset du timer
- **Image de fond** : bitmap 320×480 RGB565 portrait (fluide grâce à la PSRAM)

---

## Pinout JC3248W535C

| Fonction | GPIO | Composant ESPHome |
|---|---|---|
| QSPI CLK | 47 | `spi.clk_pin` |
| QSPI DATA | 21, 48, 40, 39 | `spi.data_pins` |
| Touch SDA | 4 | `i2c.sda` |
| Touch SCL | 8 | `i2c.scl` |
| Backlight (PWM) | 1 | `output.ledc` |

**Pas de `reset_pin`** pour le touch — le driver `axs15231` n'en a pas besoin sur ce board.

---

## Layout portrait

```
┌──────────────────────┐  y=0
│ MAISON           HC  │  y=28  (header + tarif coloré)
│ ──────────────────── │  y=42
│ * Extérieur  14.6 °C │  y=92
│ * Intérieur  21.0 °C │  y=144
│ * Piscine    19.8 °C │  y=196
│ ──────────────────── │  y=226
│ * Solaire    +1450 W │  y=278
│ * Maison      620 W  │  y=330
│ * Zoé    62% 145km   │  y=382
│ ──────────────────── │  y=430
│ MAJ 14:32            │  y=462
└──────────────────────┘  y=480
```

---

## Différences vs CYD ESP32-3248S035C

| | CYD 3248S035C | JC3248W535C |
|---|---|---|
| SoC | ESP32 | **ESP32-S3** |
| PSRAM | ❌ | ✅ **8 MB octal** |
| Flash | 4 MB | **16 MB** |
| Framework | Arduino | **ESP-IDF** |
| SPI | Standard (MOSI/CLK/CS/DC) | **QSPI quad (4 data pins)** |
| SPI CLK | GPIO 14 | GPIO 47 |
| SPI DATA | GPIO 13 (MOSI) | GPIO 21/48/40/39 |
| I2C SDA | GPIO 33 | GPIO 4 |
| I2C SCL | GPIO 32 | GPIO 8 |
| Backlight | GPIO 27 | GPIO 1 |
| Driver display | `model: ST7796` | `model: JC3248W535` |
| Driver touch | `gt911` | `axs15231` |
| Orientation | Landscape 480×320 | **Portrait 320×480 (natif)** |
| Pages | 2 pages avec tap | **1 page unifiée** |
| RAM utilisée | ~65% | **~12%** |
| Compile initiale | ~10 min | **~30 sec** (cache) |
| Image de fond | Lente (sans PSRAM) | **Fluide** |
| Réactivité tap | Correcte | **Excellente** |

---

## Pièges rencontrés

### 1. Pas de `transform:` sur le modèle JC3248W535

Contrairement au ST7796 générique, le modèle built-in `JC3248W535` dans ESPHome ne supporte pas le bloc `transform:`. Erreur typique :

```
Axis swapping not supported by this model.
```

**Solution** : utiliser `rotation:` à la place. Pour le mode portrait natif : `rotation: 0`. Pour le landscape : `rotation: 90`.

### 2. Driver touch : `axs15231`, pas `gt911`

Le touch est géré par le même chip AXS15231B que le display. Symptôme : aucun événement au tap avec `gt911`.

```yaml
touchscreen:
  - platform: axs15231
    i2c_id: bus_touch
    # Pas de reset_pin
```

### 3. Framework ESP-IDF obligatoire pour la PSRAM

```yaml
esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  flash_size: 16MB
  framework:
    type: esp-idf

psram:
  mode: octal
  speed: 80MHz
```

### 4. NaN dans les polices custom

Quand un sensor renvoie une valeur invalide, `snprintf` écrit `"nan"` ou `"inf"` dans le buffer. Si la police ne contient pas ces caractères, ESPHome log un warning et n'affiche rien.

**Solution** : ajouter `nai` aux glyphs de toute police utilisée pour des valeurs numériques :

```yaml
glyphs: "0123456789.-+ %kmnai"   # nai couvre "nan" et "inf"
```

### 5. Premier flash obligatoirement en USB-C

L'OTA n'est pas disponible sur un device vierge. Premier flash via câble USB-C.  
Si le port n'est pas détecté : **BOOT** (maintenu) + **RESET** (bref) pour forcer le mode DFU.

### 6. Clé API HA indispensable pour les sensors

Sans la bonne clé API, le device se connecte à HA mais HA ne pousse pas les valeurs. Affichage `--.-` partout.

---

## Installation

### Prérequis

- Home Assistant avec l'add-on **ESPHome** installé
- Sensors sources déjà exposés comme entités dans HA
- Polices dans `/config/esphome/fonts/` :
  - `Roboto-Regular.ttf`
  - `Roboto-Medium.ttf`
  - `materialdesignicons-webfont.ttf`
- Image de fond 320×480 dans `/config/esphome/images/` (optionnel)

### Étapes

1. **Copier** `thermometre_jc3228.yaml` dans `/config/esphome/`
2. **Adapter les `entity_id`** dans le bloc `sensor:` pour matcher tes entités HA
3. **Compléter `secrets.yaml`** avec `api_encryption_key`, `ota_password`, `wifi_ssid`, `wifi_password`
4. **Premier flash via USB-C** (ESPHome → Install → Plug into the computer)
5. **Ensuite tout passe en OTA** automatiquement

---

## Personnalisation

### Durée de veille

```yaml
substitutions:
  dim_delay: "3min"    # 30s, 2min, 5min, etc.
```

### Luminosité en veille

Dans le script `dim_timer`, changer `brightness: 60%` selon l'environnement :
- Chambre : 0% (extinction complète)
- Couloir nuit : 15-20%
- Cuisine/salon : 50-60%

### Seuils de couleurs thermiques

Dans le lambda, fonctions `color_temp_ext`, `color_temp_pool`, `color_temp_int`.

### Seuils batterie Zoé

```cpp
if (id(zoe_battery_percent).state < 20.0f) {
  zoe_col = id(c_red);       // critique
} else if (id(zoe_battery_percent).state < 40.0f) {
  zoe_col = id(c_orange);    // attention
}
// sinon c_green (normal)
```

### Changer l'image de fond

Remplacer le fichier 320×480 PNG dans `images/` et adapter le nom dans le bloc `image:`.

---

## Performances observées

| Métrique | Valeur |
|---|---|
| RAM utilisée | ~12% (40 KB / 328 KB) |
| Flash utilisée | ~12% |
| Compile initiale | ~30 sec (cache ESP-IDF) |
| Upload OTA | ~4 sec |
| Réactivité tap | < 100 ms |
| Refresh auto | toutes les 10s |
| Mise en veille | après 3 min (configurable) |
| Luminosité veille | 60% (configurable) |

---

## Fichiers

```
.
├── README.md
├── thermometre_jc3228.yaml
└── images/
    └── bg_thermometre_portrait_320x480.png
```

Les polices sont partagées avec le projet CYD dans `/config/esphome/fonts/`.

---

## Licence

MIT — libre d'utilisation, modification et redistribution.

---

## Crédits

- Config de base : [RyanEwen/esphome-lvgl](https://github.com/RyanEwen/esphome-lvgl) et [forum HA](https://community.home-assistant.io/t/jc3248w535-guition-3-5-config/791363)
- ESPHome — [esphome.io](https://esphome.io)
- Material Design Icons — [pictogrammers.com](https://pictogrammers.com/library/mdi/)
- Roboto — Google Fonts