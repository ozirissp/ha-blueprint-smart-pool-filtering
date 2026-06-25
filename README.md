# ha-blueprint-smart-pool-filtering

Blueprint Home Assistant pour la filtration intelligente de piscine par cycles. Il calcule une durée de filtration journalière à partir de la température de l'eau, la répartit en N cycles espacés uniformément sur une fenêtre horaire configurable, et pilote la pompe via un polling régulier.

## Fonctionnement

### Calcul de la durée totale

```
durée_totale (min) = température / coefficient × 60
```

Calculée une fois par jour à `calc_time`, écrite dans un `input_number` (ajustable manuellement avant l'ouverture de la fenêtre).

### Répartition en cycles

La durée totale est découpée en N cycles répartis sur `[window_start, window_end]` avec des pauses égales entre chaque cycle.

```
Exemple : 2 cycles, 6h totales, fenêtre 8h→20h

Durée/cycle = 6h / 2 = 3h
Pause       = (12h - 6h) / 1 = 6h

Cycle 1 : 08h00 → 11h00
Cycle 2 : 17h00 → 20h00
```

Le nombre de cycles est **auto-ajusté** si la durée par cycle dépasse les bornes `min/max_cycle` :
- durée/cycle > `max_cycle` → N augmente
- durée/cycle < `min_cycle` → N diminue

Si la durée totale ≥ fenêtre → 1 seul cycle couvrant toute la fenêtre.

### Déclencheurs

Le blueprint utilise 4 déclencheurs :

| ID | Déclencheur | Action |
|---|---|---|
| `calc` | `calc_time` (heure fixe) | Calcule et écrit la durée dans l'`input_number` |
| `poll` | Toutes les X minutes | Vérifie « suis-je dans un cycle ? » et pilote la pompe |
| `poll` | Redémarrage HA | Même logique (reprise après reboot) |
| `external` | Événement custom | Recalcul + réévaluation immédiate |

### Logique de polling

À chaque vérification :

```
Calculer les créneaux [start_i, end_i] depuis l'input_number
→ Si dans un créneau ET pompe arrêtée  → start_action
→ Si hors créneau ET pompe en marche   → stop_action
→ Sinon                                → rien
```

---

## Prérequis

### 1. Sensor de température

N'importe quel sensor avec `device_class: temperature`. Exemples :

```yaml
# Sensor brut
sensor.pool_temperature

# Statistics sensor (moyenne sur 24h)
sensor:
  - platform: statistics
    name: "Pool temperature 24h mean"
    entity_id: sensor.pool_temperature
    state_characteristic: mean
    max_age:
      hours: 24

# Template sensor
template:
  - sensor:
      - name: "Pool temperature smoothed"
        unit_of_measurement: "°C"
        device_class: temperature
        state: "{{ states('sensor.pool_temperature') | float(20) }}"
```

### 2. input_number de durée cible

Créer un helper **Number** via l'UI HA :

**Paramètres → Appareils et services → Entrées → Ajouter → Nombre**

| Champ | Valeur (si unité `min`) | Valeur (si unité `h`) |
|---|---|---|
| Nom | Pool filter target duration | Pool filter target duration |
| Valeur min | 0 | 0 |
| Valeur max | 720 | 12 |
| Pas | 1 | 0.5 |
| Unité | `min` | `h` |
| Icône | `mdi:timer` | `mdi:timer` |

Le blueprint détecte automatiquement l'unité (`min`, `h` ou `s`) et adapte l'écriture et la lecture.

Exemple en YAML :

```yaml
input_number:
  pool_filter_target_duration:
    name: "Pool filter target duration"
    min: 0
    max: 720
    step: 1
    unit_of_measurement: min
    icon: mdi:timer
```

### 3. Sensor d'état de la pompe

Entité dont l'état est `on` quand la pompe tourne. Peut être un `switch`, `input_boolean`, `binary_sensor` ou `sensor`.

### 4. Actions de démarrage et d'arrêt

Les actions doivent être **idempotentes** (appeler `turn_on` sur un switch déjà allumé est sans effet) :

- `homeassistant.turn_on` / `homeassistant.turn_off` sur un switch ou input_boolean
- L'appel d'un script
- L'activation d'une scène

---

## Paramètres

| Paramètre | Défaut | Description |
|---|---:|---|
| Water temperature sensor | — | Sensor de température |
| Pump state sensor | — | Entité indiquant si la pompe tourne (`on` = active) |
| Target duration input_number | — | Entité de la durée cible (unité auto-détectée) |
| Start filtration action | — | Action de démarrage de la pompe |
| Stop filtration action | — | Action d'arrêt de la pompe |
| Daily calculation time | `06:00` | Heure de calcul quotidien |
| Filtration window start | `08:00` | Début de la fenêtre de filtration |
| Filtration window end | `20:00` | Fin de la fenêtre de filtration |
| Verification interval | `5 min` | Fréquence du polling |
| Number of cycles | `2` | Nombre de cycles souhaité (ajusté automatiquement) |
| Minimum cycle duration | `60 min` | Durée minimum par cycle |
| Maximum cycle duration | `240 min` | Durée maximum par cycle |
| Filtration coefficient | `3` | Diviseur température → durée |
| External trigger event type | `smart_pool_filtering_run` | Nom de l'événement custom |

---

## Trigger externe

Forcer un recalcul + réévaluation immédiate depuis une automation ou un script :

```yaml
action: event.fire
data:
  event_type: smart_pool_filtering_run
```

Utile pour déclencher manuellement (bouton Lovelace via un script), après un changement de température, ou depuis une autre automation.

Si tu as plusieurs instances du blueprint (plusieurs piscines), utilise un `external_event_type` différent par instance.

---

## Comportements notables

- La durée cible est calculée à `calc_time` et ajustable manuellement avant `window_start`.
- Le blueprint détecte automatiquement l'unité de l'`input_number` (`min`, `h`, `s`).
- Le nombre de cycles est ajusté automatiquement pour respecter `min/max_cycle`.
- Si durée totale ≥ fenêtre : 1 cycle couvrant toute la fenêtre `[window_start, window_end]`.
- Si la pompe s'arrête seule dans un cycle, elle sera relancée au prochain poll.
- Si HA redémarre, le blueprint réévalue immédiatement le créneau actif.
- `mode: single` + `max_exceeded: silent` évitent les instances parallèles.
- La fenêtre doit être sur le même jour (pas de créneau passant minuit).

---

## Installation

### Import direct depuis l'UI HA (recommandé)

1. Dans HA, aller dans **Paramètres → Automatisations → Blueprints**
2. Cliquer sur **Importer un blueprint** (bouton en bas à droite)
3. Coller l'URL :
   ```
   https://raw.githubusercontent.com/ozirissp/ha-blueprint-smart-pool-filtering/main/smart_pool_filtering.yaml
   ```
4. Cliquer **Aperçu** puis **Importer**

Créer une automation depuis **Paramètres → Automatisations → Créer une automatisation → Depuis un blueprint**.

### Manuellement

Copier `smart_pool_filtering.yaml` dans :

```
config/blueprints/automation/smart_pool_filtering/smart_pool_filtering.yaml
```

Puis **Paramètres → Automatisations → Blueprints → ⋮ → Recharger les blueprints**.

> **Mise à jour :** supprimer l'ancien blueprint via l'UI et réimporter depuis l'URL. Les automatisations existantes devront être recréées.
