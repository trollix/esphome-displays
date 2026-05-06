# Dashboard CYD — JC3248W535C pour Home Assistant

Firmware ESPHome pour le **JC3248W535C** — ESP32-S3 3.5" capacitif, 8 MB PSRAM.  
Dashboard tactile affichant températures et données énergie, connecté à Home Assistant.

Évolution du projet [thermometre-cyd](../thermometre-cyd/README.md) (ESP32-3248S035C), porté sur ESP32-S3 avec PSRAM pour une réactivité nettement améliorée.

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

- **2 pages** switchées par tap sur l'écran
- **Page 1 — Températures** : extérieur, intérieur, piscine avec couleurs contextuelles selon les seuils
- **Page 2 — Énergie** : tarif HP/HC dans le header, production solaire, consommation maison, charge Zoé (% + km)
- **Veille automatique** : dim à 15% après 3 minutes d'inactivité (configurable via substitution)
- **Réveil au tap** : premier tap = réveil sans changer de page, tap suivant = changement de page
- **Image de fond** : bitmap 480×320 RGB565 (possible sans pénalité grâce à la PSRAM)

---

## Pinout JC3248W535C

| Fonction | GPIO | Composant ESPHome |
|---|---|---|
| QSPI CLK | 47 | `spi.clk_pin` |
| QSPI DATA | 21, 48, 40, 39 | `spi.data_pins` |
| Touch SDA | 4 | `i2c.sda` |
| Touch SCL | 8 | `i2c.scl` |
| Backlight (PWM) | 1 | `output.ledc` |

⚠️ **Pas de `reset_pin`** pour le touch — le driver `axs15231` n'en a pas besoin sur ce board.

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
| RAM utilisée | ~65% | **~12%** |
| Compile initiale | ~10 min | **~30 sec** (cache) |
| Image de fond | Lente (sans PSRAM) | **Fluide** |
| Réactivité tap | Correcte | **Excellente** |

**Le lambda, les couleurs, les polices, les sensors HA — 100% identiques** entre les deux boards. Seuls les 3-4 blocs hardware changent.

---

## Pièges rencontrés

### 1. Pas de `transform:` sur le modèle JC3248W535

Contrairement au ST7796 générique, le modèle built-in `JC3248W535` dans ESPHome ne supporte pas le bloc `transform:` (swap_xy, mirror_x, mirror_y). Erreur typique :

```
Axis swapping not supported by this model.
```

**Solution** : utiliser `rotation: 90` à la place du bloc `transform:`.

### 2. Driver touch : `axs15231`, pas `gt911`

Le touch est géré par le même chip AXS15231B que le display (contrairement au CYD qui a un GT911 séparé). Symptôme : aucun événement au tap avec `gt911`.

```yaml
touchscreen:
  - platform: axs15231
    i2c_id: bus_touch
    # Pas de reset_pin
```

### 3. Framework ESP-IDF obligatoire pour la PSRAM

Le mode Arduino ne gère pas bien la PSRAM octal sur ESP32-S3. Utiliser ESP-IDF.

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

### 4. Premier flash obligatoirement en USB-C

L'OTA n'est pas disponible sur un device vierge. Premier flash via câble USB-C branché sur le PC faisant tourner ESPHome.

Sur certains boards, appuyer sur **BOOT** (maintenu) + **RESET** (bref) pour forcer le mode DFU si le port n'est pas détecté.

### 5. Clé API HA indispensable pour les sensors

Sans la bonne clé API dans le YAML, le device se connecte à HA mais HA ne pousse pas les valeurs des `homeassistant:` sensors. Le device affiche `--.-` partout.

S'assurer que la clé dans le YAML correspond à celle déclarée lors de l'adoption du device dans HA.

---

## Installation

### Prérequis

- Home Assistant avec l'add-on **ESPHome** installé
- Sensors sources déjà exposés comme entités dans HA
- Polices dans `/config/esphome/fonts/` :
  - `Roboto-Regular.ttf`
  - `Roboto-Medium.ttf`
  - `Roboto-SemiBold.ttf` *(optionnel selon config)*
  - `materialdesignicons-webfont.ttf`
- Image de fond dans `/config/esphome/images/` (optionnel)

### Étapes

1. **Copier** `thermometre-jc3248.yaml` dans `/config/esphome/`
2. **Adapter les `entity_id`** dans le bloc `sensor:` pour matcher tes entités HA
3. **Compléter `secrets.yaml`** avec `api_encryption_key`, `ota_password`, `wifi_ssid`, `wifi_password`
4. **Premier flash via USB-C** (ESPHome → Install → Plug into the computer)
5. **Ensuite tout passe en OTA** automatiquement

---

## Personnalisation

### Durée de veille

Changer la valeur en haut du fichier :

```yaml
substitutions:
  dim_delay: "3min"    # 30s, 2min, 5min, etc.
```

### Seuils de couleurs thermiques

Dans le lambda, fonctions `color_temp_ext`, `color_temp_pool`, `color_temp_int`.

### Changer l'image de fond

Remplacer le fichier dans `images/` et adapter le nom dans le bloc `image:`.  
Pour générer une image optimisée RGB565 avec dithering, voir le README principal du projet.

### Ajouter une page

1. Modifier `% 2` en `% 3` dans `on_touch:`
2. Ajouter un `else if (id(current_page) == 2)` dans le lambda
3. Ajouter un 3e cercle dans `draw_dots()`

---

## Performances observées

| Métrique | Valeur |
|---|---|
| RAM utilisée | ~12% (40 KB / 328 KB) |
| Flash utilisée | ~12% (991 KB / 8 MB) |
| Compile initiale | ~30 sec (cache ESP-IDF) |
| Upload OTA | ~4 sec |
| Tap → changement de page | < 100 ms |
| Refresh auto | toutes les 10s |
| Mise en veille | après 3 min (configurable) |

---

## Fichiers

```
.
├── README.md
├── thermometre-jc3248.yaml
└── images/
    └── bg_thermometre_cyd_480x320.png    # image de fond (optionnelle)
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