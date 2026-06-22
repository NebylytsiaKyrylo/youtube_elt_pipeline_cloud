# Naming Conventions

Ce document couvre uniquement les décisions propres à ce projet.
Les standards universels (snake_case, mots réservés SQL) ne sont pas répétés ici.

---

## Schemas

| Schema         | Rôle                                              | Matérialisation | Exemple                                   |
|----------------|---------------------------------------------------|-----------------|-------------------------------------------|
| `staging`      | Miroir TEXT de la source — aucune transformation  | `view`          | `stg_yt_video_snapshot`                   |
| `intermediate` | Casts de types, renommage — aucune logique métier | `view`          | `int_yt_video_snapshot`                   |
| `core`         | Star schema — dimensions et faits (SCD Type 1)    | `incremental`   | `dim_channel`, `fct_video_daily_snapshot` |
| `marts`        | Tables dénormalisées pour Superset                | `table`         | `mart_channel_top_views`                  |

---

## Tables

| Couche         | Pattern                        | Exemple                                                  |
|----------------|--------------------------------|----------------------------------------------------------|
| `staging`      | `stg_<source>_<entity>`        | `stg_yt_video_snapshot`                                  |
| `intermediate` | `int_<source>_<entity>`        | `int_yt_video_snapshot`                                  |
| `core`         | `dim_<entity>`, `fct_<entity>` | `dim_channel`, `fct_video_daily_snapshot`                |
| `marts`        | `mart_<domain>_<metric>`       | `mart_channel_top_views`, `mart_video_format_engagement` |

`<source>` = `yt` — abréviation retenue pour YouTube (pas `youtube`, trop long dans les noms de tables).

---

## Colonnes

| Pattern / Suffixe | Usage                             | Exemple                           |
|-------------------|-----------------------------------|-----------------------------------|
| `<entity>_key`    | Surrogate key (core uniquement)   | `channel_key`, `video_key`        |
| `<entity>_id`     | Business key (identifiant source) | `video_id`, `channel_id`          |
| `dwh_<name>`      | Métadonnée système                | `dwh_loaded_at`, `dwh_updated_at` |
| `is_<state>`      | Boolean flag                      | `is_active`                       |
| `_at`             | Timestamp d'événement             | `published_at`, `deleted_at`      |
| `_date`           | Date calendaire                   | `snapshot_date`                   |

**Colonnes `dwh_` — présentes uniquement dans `core` :**

| Colonne          | Type           | Tables                                                           |
|------------------|----------------|------------------------------------------------------------------|
| `dwh_loaded_at`  | `TIMESTAMP_TZ` | Toutes les tables `core`                                         |
| `dwh_updated_at` | `TIMESTAMP_TZ` | `dim_channel`, `dim_video` uniquement — les faits sont immuables |

**Soft delete dans `dim_video` :** `is_active` (BOOLEAN) + `deleted_at` (TIMESTAMP_TZ).

---

## Fichiers

### Python

| Pattern                  | Rôle                              | Exemple                                    |
|--------------------------|-----------------------------------|--------------------------------------------|
| `<noun>.py`              | Module — nommé par responsabilité | `client.py`, `loader.py`, `extractor.py`   |
| `<source>_<type>_dag.py` | DAG Airflow                       | `yt_elt_dag.py`                            |
| `test_<module>.py`       | Test unitaire                     | `test_loader.py`, `test_client.py`         |
| `email_<event>.html`     | Template de notification          | `email_failure.html`, `email_success.html` |

### dbt

Les fichiers modèles suivent le même pattern que les tables (ex: `stg_yt_video_snapshot.sql`).
Les fichiers de configuration (`dbt_project.yml`, `profiles.yml`, `sources.yml`, `schema.yml`, `packages.yml`) suivent
les noms imposés par dbt.
