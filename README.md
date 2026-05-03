# ha-blueprint-smart-pool-filtering

Blueprint Home Assistant pour la filtration intelligente de piscine. Calcule automatiquement la durée de filtration journalière à partir de la température moyenne sur 24h (règle du tiers configurable). Entièrement configurable via l'UI HA.

## Fonctionnement

```
durée_brute   = température_24h / coefficient
durée_finale  = max(durée_min, min(durée_max, durée_brute))
```

À l'heure de démarrage configurée, l'automation :

1. Vérifie si le quota journalier est déjà atteint (optionnel)
2. Exécute l'**action de démarrage** de la pompe
3. Attend la durée calculée (ou la durée restante si un sensor est configuré)
4. Exécute l'**action d'arrêt** de la pompe

## Prérequis

### 1. Statistics sensor (obligatoire)

Créer un sensor de statistiques natif HA pointant sur le capteur de température, configuré en `mean` sur 24h. C'est ce sensor qui est sélectionné dans le champ **"Temperature sensor"**.

```yaml
# configuration.yaml ou via l'UI
sensor:
  - platform: statistics
    name: "Pool temperature 24h mean"
    entity_id: sensor.pool_temperature
    state_characteristic: mean
    max_age:
      hours: 24
```

### 2. Actions de démarrage et d'arrêt (obligatoires)

Préparer une action pour démarrer la pompe et une pour l'arrêter. Il peut s'agir de :
- `homeassistant.turn_on` / `turn_off` sur un switch ou input_boolean
- L'appel d'un script
- L'activation d'une scène

### 3. Sensor "déjà filtré aujourd'hui" (optionnel)

Un template ou un sensor retournant le nombre de **minutes** déjà filtrées aujourd'hui. Si la durée calculée est déjà atteinte, la filtration ne démarre pas.

Exemple avec un `history_stats` sensor :

```yaml
sensor:
  - platform: history_stats
    name: "Pool filter today minutes"
    entity_id: switch.pool_pump
    state: "on"
    type: time
    start: "{{ now().replace(hour=0, minute=0, second=0) }}"
    end: "{{ now() }}"
```

Le template à saisir dans le blueprint :

```
{{ (states('sensor.pool_filter_today_minutes') | float(0) * 60) | round(0) }}
```

*(history_stats retourne des heures → multiplier par 60 pour obtenir des minutes)*

## Paramètres

| Paramètre | Défaut | Description |
|---|---|---|
| Temperature sensor | — | Statistics sensor (mean 24h) |
| Start filtration action | — | Action HA pour démarrer la pompe |
| Stop filtration action | — | Action HA pour arrêter la pompe |
| Filtration start time | 10:00 | Heure de déclenchement quotidien |
| Filtration coefficient | 3 | Diviseur température → durée |
| Minimum duration | 2h | Plancher (sensor indisponible → min) |
| Maximum duration | 12h | Plafond de filtration |
| Already filtered today | 0 | Template Jinja2 (minutes déjà filtrées) |

## Comportements notables

- **Sensor indisponible** (`unknown`/`unavailable`) → durée tombe à `durée_min`
- **Durée calculée < min** → clampée à `durée_min`
- **Durée calculée > max** → clampée à `durée_max`
- **Quota déjà atteint** → l'automation s'arrête sans démarrer la pompe
- **`mode: single`** → pas d'instances parallèles, `max_exceeded: silent`

## Installation

### Via HACS

Ajouter ce dépôt comme dépôt personnalisé dans HACS (type : Automation).

### Manuellement

Copier `smart_pool_filtering.yaml` dans :

```
config/blueprints/automation/smart_pool_filtering/
```

Puis **Paramètres → Automatisations → Blueprints → Recharger les blueprints**.
