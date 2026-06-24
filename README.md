# ha-blueprint-smart-pool-filtering

Blueprint Home Assistant pour la filtration intelligente de piscine. Il calcule une durée cible journalière à partir de la température de l'eau, l'écrit dans un `input_number`, démarre la filtration à une heure configurable, puis vérifie régulièrement un sensor d'historique pour arrêter la pompe quand le quota est atteint.

## Fonctionnement

La durée cible est calculée une fois par jour :

```
durée_brute   = température / coefficient
durée_finale  = max(durée_min, min(durée_max, durée_brute))
```

Le blueprint utilise trois déclencheurs :

1. **Calcul** à `calc_time` : calcule la durée cible et l'écrit dans l'`input_number` en minutes.
2. **Démarrage** à `start_time` : exécute l'action de démarrage de la pompe.
3. **Surveillance** toutes les X minutes : lit le sensor d'historique et arrête la pompe si `durée_filtrée >= durée_cible`.

Entre l'heure de calcul et l'heure de démarrage, l'utilisateur peut modifier manuellement l'`input_number` pour ajuster la durée du jour.

## Prérequis

### 1. Sensor de température

N'importe quel sensor retournant une température de l'eau avec `device_class: temperature`. Il peut s'agir d'un sensor brut, d'un sensor de statistiques, ou d'un template sensor.

Exemples :

```yaml
# Sensor brut (directement depuis l'intégration de la sonde)
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

Créer un helper `input_number` qui stocke la durée cible en minutes.

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

### 3. Sensor d'historique

Créer un sensor qui retourne la durée filtrée aujourd'hui. Avec `history_stats`, `type: time` retourne des **heures**.

```yaml
sensor:
  - platform: history_stats
    name: "Pool filter today"
    entity_id: switch.pool_pump
    state: "on"
    type: time
    start: "{{ today_at() }}"
    end: "{{ now() }}"
```

Le blueprint détecte automatiquement l'unité via l'attribut `unit_of_measurement` du sensor (fallback sur `h` si absent). Aucune configuration supplémentaire requise.

### 4. Actions de démarrage et d'arrêt

Préparer une action pour démarrer la pompe et une pour l'arrêter. Il peut s'agir de :

- `homeassistant.turn_on` / `homeassistant.turn_off` sur un switch ou input_boolean
- L'appel d'un script
- L'activation d'une scène

## Paramètres

| Paramètre | Défaut | Description |
|---|---:|---|
| Water temperature sensor | — | Sensor de température (raw, statistics, template…) |
| Filtered duration today sensor | — | Sensor d'historique de filtration du jour |
| Target duration input_number | — | Entité où écrire/lire la durée cible en minutes |
| Start filtration action | — | Action HA pour démarrer la pompe |
| Stop filtration action | — | Action HA pour arrêter la pompe |
| Daily calculation time | `06:00` | Heure de calcul de la durée cible |
| Filtration start time | `10:00` | Heure de démarrage de la filtration |
| Verification interval | `15 min` | Fréquence de vérification du quota |
| Filtration coefficient | `3` | Diviseur température → durée |
| Minimum duration | `2h` | Durée minimum |
| Maximum duration | `12h` | Durée maximum |

## Comportements notables

- La durée cible est calculée uniquement à `calc_time`.
- Le démarrage à `start_time` ne recalcule pas la durée : il utilise la valeur actuelle de l'`input_number`.
- Si la pompe s'arrête toute seule avant quota, le blueprint ne la redémarre pas.
- Le blueprint appelle `stop_action` quand le sensor d'historique atteint ou dépasse la durée cible.
- Le polling utilise un trigger toutes les minutes, filtré par `Verification interval`.
- `mode: single` + `max_exceeded: silent` évitent les instances parallèles et le spam de logs.

## Installation

### Via HACS

Ajouter ce dépôt comme dépôt personnalisé dans HACS (type : Automation).

### Manuellement

Copier `smart_pool_filtering.yaml` dans :

```
config/blueprints/automation/smart_pool_filtering/
```

Puis **Paramètres → Automatisations → Blueprints → Recharger les blueprints**.
