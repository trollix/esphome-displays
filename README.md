# Thermomètre CYD — Dashboard tactile pour Home Assistant

Petit afficheur tactile autonome basé sur un **ESP32-3248S035C** (alias *Cheap Yellow Display* 3.5" capacitif), piloté par **ESPHome** et connecté à **Home Assistant**.

Affiche en deux pages :
- **Températures** : extérieur, intérieur, piscine
- **Énergie** : tarif HP/HC (Lixee/Téléinfo), production solaire nette, consommation maison

Navigation par **tap sur l'écran**.

![exemple d'affichage](docs/screenshot.jpg)

---

## Matériel

- **Carte** : ESP32-3248S035**C** (*ESP32 + écran 3.5" 320×480 + tactile capacitif GT911*) — version capacitive uniquement (pas la R résistive)
- **Alimentation** : chargeur 5V/2A via micro-USB (⚠️ **pas un port USB de PC**, instabilité possible)
- **Boîtier** : acrylique fourni avec la carte chez DIYmalls/AITRIP/etc.

Compter ~25-30 € sur Amazon, ~15-20 € sur AliExpress.

---

## Architecture

```
Capteurs HA  ─┐
              ├─►  Home Assistant  ─►  API ESPHome  ─►  ESP32-3248S035C  ─►  Écran 3.5"
   (Lixee,    │
    Ecowitt,  │
    sonde     │
    piscine)  ┘
```

L'ESP ne capte rien lui-même : il consomme uniquement des `sensor.xxx` déjà exposés dans HA. La logique métier reste côté HA, le device est purement un afficheur tactile.

---

## Pinout ESP32-3248S035C

| Fonction | GPIO | Composant ESPHome |
|---|---|---|
| TFT CLK | 14 | `spi.clk_pin` |
| TFT MOSI | 13 | `spi.mosi_pin` |
| TFT CS | 15 | `display.cs_pin` |
| TFT DC | 2 | `display.dc_pin` |
| Backlight (PWM) | 27 | `output.ledc` |
| Touch SDA | 33 | `i2c.sda` |
| Touch SCL | 32 | `i2c.scl` |
| Touch RST | 25 | `touchscreen.reset_pin` |

⚠️ **Pas d'`interrupt_pin`** pour le touch — bug HW connu sur le 3248S035C (INT du GT911 connecté à GND par erreur).

---

## Pièges rencontrés et solutions

Ce projet a généré quelques heures de debug. Voici les écueils principaux pour épargner le suivant :

### 1. Driver display : `mipi_spi`, pas `ili9xxx`

Le composant historique `ili9xxx` essaie d'allouer un buffer complet (480×320×2 = 307 KB) qui ne tient pas en RAM sans PSRAM. Erreur typique :
```
[E][component:224]: display is marked FAILED: unspecified
[ili9xxx:124]: Failed to init Memory: YES!
```

**Solution** : utiliser le composant moderne `mipi_spi` qui gère le partial buffer.

```yaml
display:
  - platform: mipi_spi
    model: ST7796
    color_order: bgr
    draw_rounding: 1
    auto_clear_enabled: true
    dimensions:
      width: 480
      height: 320
    transform:
      swap_xy: true
      mirror_x: false
      mirror_y: false
```

### 2. Driver touch : `gt911`, pas `ft63x6`

Le board utilise une puce **GT911** (et non FT6336U comme on pourrait le croire au premier abord). Symptôme : aucun event au tap, aucune ligne dans les logs.

```yaml
touchscreen:
  - platform: gt911
    reset_pin: GPIO25
    # Pas d'interrupt_pin — bug HW
```

### 3. Modèle d'écran : `ST7796`

Certaines docs annoncent un ILI9488. Sur ce board c'est en fait un ST7796 (compatible ILI9488 à 95%, mais la séquence d'init diffère).

### 4. Color order : `bgr` + couleurs RGB normales

Avec `color_order: bgr` dans le YAML et des définitions RGB classiques dans le bloc `color:`, on obtient le bon rendu. Si on change `color_order` ou si on swap manuellement R↔B, on tombe dans des combinaisons bizarres (fond blanc, couleurs inversées…).

### 5. Image de fond : à éviter sans PSRAM

Une image bitmap 480×320 RGB565 fait 307 KB en flash et ralentit énormément le rafraîchissement (~300 ms par frame vs ~50 ms en fond uni). **Le tap devient mou.**

Mieux vaut un fond uni ou des formes simples dessinées en lambda (`it.fill()`, `it.line()`, `it.filled_rectangle()`).

### 6. `update_interval` ≠ réactivité du tap

`update_interval` règle la fréquence de rafraîchissement automatique. Le tap, lui, force un refresh immédiat via `component.update: disp` dans `on_touch:`. Donc on peut sans souci rester à `update_interval: 10s` (économe en CPU) tout en ayant un tap instantané.

---

## Installation

### Prérequis

- Home Assistant avec l'add-on **ESPHome** installé
- Capteurs sources déjà exposés en `sensor.xxx` dans HA

### Étapes

1. **Cloner ce repo** dans `/config/esphome/` (ou copier les fichiers)
2. **Placer les polices** dans `/config/esphome/fonts/` :
   - `Roboto-Regular.ttf`
   - `Roboto-Medium.ttf`
   - `Roboto-SemiBold.ttf`
   - `materialdesignicons-webfont.ttf`
3. **Adapter les `entity_id`** dans le bloc `sensor:` du YAML pour matcher tes capteurs HA
4. **Compléter `secrets.yaml`** avec `wifi_ssid`, `wifi_password`
5. **Premier flash via micro-USB** (depuis l'add-on ESPHome → Install → Plug into the computer running ESPHome Dashboard)
6. **Ensuite tout passe en OTA** sur le réseau

Première compile longue (~10 min), suivantes incrémentales (~30 s).

---

## Personnalisation

### Changer les seuils de couleurs thermiques

Dans le lambda, fonctions `color_temp_ext`, `color_temp_pool`, `color_temp_int`. Adapter les seuils selon ton climat et tes préférences.

### Changer les icônes

Codepoints MDI au début du lambda comme constantes :
```cpp
const char* ICON_SUN   = "\U000F0599"; // mdi:weather-sunny
```

Pour trouver d'autres icônes : [pictogrammers.com/library/mdi](https://pictogrammers.com/library/mdi/) → cliquer sur l'icône → noter le codepoint hex (`F0599` → `\U000F0599` dans le YAML).

Penser à ajouter le glyph dans la liste `glyphs:` du bloc `font:`.

### Ajouter une page

Modifier `% 2` en `% 3` dans le `on_touch:`, ajouter un bloc `else if (id(current_page) == 1)` dans le lambda, et incrémenter le nombre de points dans `draw_dots`.

---

## Fichiers du repo

```
.
├── README.md                          # ce fichier
├── thermometre-cyd.yaml               # config ESPHome principale
├── secrets.yaml.example               # template à renommer en secrets.yaml
└── fonts/
    ├── Roboto-Regular.ttf
    ├── Roboto-Medium.ttf
    ├── Roboto-SemiBold.ttf
    └── materialdesignicons-webfont.ttf
```

---

## Performances observées

- **Tap → changement de page** : ~100-200 ms
- **Compilation initiale** : ~10 min (téléchargement toolchain ESP-IDF)
- **Compilations suivantes** : ~30 s à 2 min
- **Upload OTA** : ~5 s
- **Consommation RAM** : confortable, mais sans marge pour ajouter un bitmap
- **Consommation Flash** : ~70%

---

## Limites connues

- **ESP32 classique sans PSRAM** : ne pas espérer du LVGL fluide ou des animations. Pour un usage thermomètre/dashboard statique, c'est largement suffisant. Pour un projet plus ambitieux (console tactile multi-pages, animations), passer sur **ESP32-S3 + PSRAM** (JC3248W535 ou Waveshare ESP32-S3-Touch-LCD-4.3).
- **Image de fond bitmap** = ralentissement notable. Préférer un fond uni ou des formes simples dessinées en lambda.

---

## Licence

MIT — fais-en ce que tu veux, mais une mention serait sympa si ça t'aide.

---

## Crédits

- ESPHome — [esphome.io](https://esphome.io)
- Material Design Icons — [pictogrammers.com](https://pictogrammers.com/library/mdi/)
- Roboto — Google Fonts
- Inspiré du travail communautaire autour du *Cheap Yellow Display*
