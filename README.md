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

Créer un helper **Number** via l'UI HA :

**Paramètres → Appareils et services → Entrées → Ajouter → Nombre**

| Champ | Valeur |
|---|---|
| Nom | Pool filter target duration |
| Valeur min | 0 |
| Valeur max | 720 (si unité `min`) / 12 (si `h`) |
| Pas | 1 (si `min`) / 0.5 (si `h`) |
| Unité | `min`, `h` ou `s` |
| Icône | `mdi:timer` |

Le blueprint détecte automatiquement l'unité déclarée sur ce helper via l'attribut `unit_of_measurement` et adapte :

- La **valeur écrite** lors du calcul (ex. 120 → `2.0` si unité `h`)
- La **valeur lue** lors de la vérification du quota (ex. `2.0h` → 120 min en interne)

Les unités supportées sont `min` (défaut), `h` et `s`.

Exemple en YAML (`configuration.yaml`) avec unité en minutes :

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

Exemple avec unité en heures :

```yaml
input_number:
  pool_filter_target_duration:
    name: "Pool filter target duration"
    min: 0
    max: 12
    step: 0.5
    unit_of_measurement: h
    icon: mdi:timer
```

### 3. Sensor d'historique (history_stats)

Ce sensor mesure combien de temps la pompe a tourné aujourd'hui. Le blueprint lit sa valeur à chaque vérification et arrête la filtration quand le quota est atteint.

Ajouter dans `configuration.yaml` (remplacer `switch.pool_pump` par l'entité qui représente ta pompe) :

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

Puis redémarrer HA ou recharger les entités de configuration YAML.

> Le blueprint détecte automatiquement l'unité du sensor via l'attribut `unit_of_measurement` (`h` pour `history_stats type: time`). Aucune configuration supplémentaire requise.

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
| Target duration input_number | — | Entité où écrire/lire la durée cible (unité détectée automatiquement : `min`, `h` ou `s`) |
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
- Le blueprint détecte automatiquement l'unité de l'`input_number` (`min`, `h`, `s`) et adapte l'écriture et la lecture en conséquence.
- Le démarrage à `start_time` ne recalcule pas la durée : il utilise la valeur actuelle de l'`input_number`.
- Si la pompe s'arrête toute seule avant quota, le blueprint ne la redémarre pas.
- Le blueprint appelle `stop_action` quand le sensor d'historique atteint ou dépasse la durée cible.
- Le polling utilise un trigger toutes les minutes, filtré par `Verification interval`.
- `mode: single` + `max_exceeded: silent` évitent les instances parallèles et le spam de logs.

## Installation

### Import direct depuis l'UI HA (recommandé)

1. Dans HA, aller dans **Paramètres → Automatisations → Blueprints**
2. Cliquer sur **Importer un blueprint** (bouton en bas à droite)
3. Coller l'URL du fichier raw GitHub :
   ```
   https://raw.githubusercontent.com/ozirissp/ha-blueprint-smart-pool-filtering/main/smart_pool_filtering.yaml
   ```
4. Cliquer **Aperçu** puis **Importer**

Le blueprint est maintenant disponible dans la liste. Créer une automation depuis **Paramètres → Automatisations → Créer une automatisation → Depuis un blueprint**.

### Manuellement

Copier `smart_pool_filtering.yaml` dans :

```
config/blueprints/automation/smart_pool_filtering/smart_pool_filtering.yaml
```

Puis **Paramètres → Automatisations → Blueprints → ⋮ → Recharger les blueprints**.

> **Mise à jour :** pour récupérer une nouvelle version du blueprint, supprimer l'ancien via l'UI et réimporter depuis l'URL. Les automatisations existantes devront être recréées.
