# AMD GPU Stats — intégration Home Assistant

Une intégration Home Assistant custom qui affiche les **statistiques d'un GPU/APU
AMD** comme capteurs, prêts à l'emploi avec des cartes `gauge` Lovelace
(pourcentages 0–100, température 0–90 °C, puissance en watts, mémoire en GiB).

Elle ne parle pas au GPU directement. Elle poll un petit daemon compagnon —
[`AMD_GPU_Metric_DEAMON`](https://github.com/thomasbidou/AMD_GPU_Metric_DEAMON) —
qui tourne sur la machine qui a le GPU AMD + `rocm-smi`, et sert les métriques en
JSON sur le LAN.

```
┌───────────────┐   HTTP GET /   ┌────────────────────┐
│  machine AMD  │  ◄──────────── │  machine Home      │
│  rocm-smi     │   JSON         │  Assistant         │
│  daemon:8791  │ ─────────────► │  (ce repo)         │
└───────────────┘                └────────────────────┘
```

Fonctionne aussi avec les **APU** (Ryzen AI / Strix Halo, iGPU Radeon partagée)
— c'est le cas d'usage principal ici : la « VRAM » est la tranche de mémoire
unifiée adressable par le GPU, lue via `amd-smi`.

## Ce qu'elle crée

Un **appareil par GPU** (nommé d'après le GPU, ex. **Radeon 8060S Graphics on
`llm`**) plus un **appareil système** (CPU + RAM de la machine). Les capteurs et
appareils sont nommés par **UUID du GPU** et **hostname de la machine**, donc
plusieurs GPU, plusieurs machines ne se percutent jamais.

Par GPU :

| Capteur | Device class | Notes |
|---|---|---|
| Utilisation | `percentage` | 0–100 |
| Mémoire (%) | `percentage` | 0–100 |
| Mémoire (GiB) | `GiB` | valeur sur max |
| Mémoire totale | `GiB` | ta VRAM (ou tranche unifiée) |
| Consommation | `power` (W) | valeur sur max |
| Plafond | `power` (W) | `null` sur APU (pas de TDP exposé) |
| Puissance (%) | `percentage` | draw / limit — `null` si pas de limit |
| Température | `temperature` (°C) | jauge 0–90 |
| Ventilateur | `percentage` | `null` sur iGPU passive |

Sur l'appareil système (CPU + RAM de la machine) :

| Capteur | Device class | Notes |
|---|---|---|
| CPU (%) | `percentage` | 0–100 |
| Température CPU | `temperature` (°C) | capteur thermique de la machine |
| RAM (GiB) | `GiB` | valeur sur total |
| RAM totale | `GiB` | ta RAM plafond |
| RAM (%) | `percentage` | used / total |

### Carte Lovelace incluse

L'intégration embarque une carte prête à l'emploi —
`custom:amd-gpu-card` — qui montre le GPU et le CPU/RAM **en une carte**,
groupés par appareil, en liste de barres de progression. Chaque métrique a une
icône, son libellé, sa valeur et une barre colorée selon le niveau (bleu → vert
→ jaune → rouge). Elle est installée automatiquement avec l'intégration (aucun
resource manuel) et **détecte tes capteurs par leur attribut `amd_gpu_key`**,
donc elle fonctionne sur n'importe quelle machine et n'importe quel nombre de
GPU sans hard-coder d'ID d'entité.

Pour l'utiliser, pose ça dans une vue :

```yaml
type: custom:amd-gpu-card
title: llm — GPU & CPU
```

C'est tout — la carte trouve les bons capteurs toute seule. Voir
[`examples/lovelace_gauges.yaml`](examples/lovelace_gauges.yaml) pour des
jauge natives en alternative.

### Carte GPU seule

Pour n'afficher que le GPU (sans section CPU/RAM), ajoute `section: gpu` :

```yaml
type: custom:amd-gpu-card
title: llm — GPU
section: gpu
```

### Carte CPU/RAM seule

```yaml
type: custom:amd-gpu-card
title: llm — CPU & RAM
section: cpu
```

(`section: all` — défaut — affiche les deux.)

## Prérequis

- Home Assistant (core 2024+/2025+ récent).
- Le daemon compagnon lancé et joignable sur le LAN.
- `aiohttp` — déjà présent sur toute installe HA ; déclaré comme requirement
  pour que HA le résolve s'il manque.

## Installation

### 1. Lancer le daemon sur la machine AMD

Cloner et démarrer
[`AMD_GPU_Metric_DEAMON`](https://github.com/thomasbidou/AMD_GPU_Metric_DEAMON).
Adresse par défaut : `http://<machine-amd-ip>:8791/`.

### 2. Ajouter cette intégration à Home Assistant

Option A — **copier les fichiers** (ça marche partout) :

```bash
# sur la machine HA, dans le dossier de config (HAOS : /homeassistant)
git clone https://github.com/thomasbidou/AMD_GPU_Metric.git /tmp/amdgpu
cp -r /tmp/amdgpu/amd_gpu /homeassistant/custom_components/amd_gpu
```

Option B — **HACS** : ajouter ce repo comme custom repo
(`thomasbidou/AMD_GPU_Metric`), puis installer **AMD GPU Stats**.

Ensuite **redémarrer Home Assistant** — les intégrations custom ne sont
détectées qu'au boot.

### 3. Configurer

**Réglages → Appareils et services → Ajouter une intégration → « AMD GPU
Stats »**. Saisir le **host** (IP/hostname de la machine AMD) et le **port**
(défaut `8791`). Le config flow probe l'endpoint et affiche le nom du GPU sur
succès.

## Limites APU / iGPU

- **Ventilateur** : `null` sur iGPU passive (pas de ventilo) — la carte
  affiche « — ».
- **Plafond de puissance / Puissance (%)** : `null` si l'APU n'expose pas de
  TDP fixe (cas Strix Halo). La consommation (W) est bien fournie.
- **Mémoire** : `memory_used_gib`/`total_gib` = la tranche de mémoire unifiée
  adressable par le GPU (via `amd-smi`), distincte de la RAM système
  (`cpu.ram_*`).

## Troubleshooting

- **L'intégration n'apparaît pas après l'installe** → redémarrer HA ; c'est
  détecté au boot seulement.
- **Le setup échoue avec « cannot reach »** → vérifier que le daemon tourne
  (`curl http://<machine-amd-ip>:8791/health`), le port, et que la machine HA
  peut joindre la machine AMD (LAN/VLAN).
- **Les capteurs sont `unavailable`** → le daemon tourne mais `rocm-smi` a
  échoué ; regarder `rocm-smi` sur la machine AMD et les logs du daemon.

## Fichiers

```
amd_gpu/
  __init__.py        # setup/unload, construit le coordinator
  const.py           # domain + défauts
  coordinator.py     # coordinator de poll HTTP
  config_flow.py     # flow de setup UI (host + port)
  sensor.py          # les capteurs GPU + CPU/RAM (par GPU)
  card.py            # embarque + registre la carte Lovelace incluse
  www/amd-gpu-card.js  # la custom card (custom:amd-gpu-card)
  manifest.json      # manifeste de l'intégration
  strings.json       # strings utilisateur
  translations/en.json
  translations/fr.json
examples/
  lovelace_gauges.yaml
```

## Licence

MIT.
