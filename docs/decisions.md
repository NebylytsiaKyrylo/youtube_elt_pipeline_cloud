# Décisions d'architecture et d'ingénierie

Ce document consigne les choix techniques faits sur ce projet et pourquoi.

---

### D1 — Snowflake comme warehouse analytique

J'ai choisi Snowflake comme warehouse cloud. Snowflake sépare le compute du storage : le virtual warehouse s'allume uniquement quand il exécute du SQL et s'éteint automatiquement après quelques minutes d'inactivité — aucun coût d'idle. Le stockage colonnaire est natif sans configuration. `COPY INTO` permet de charger directement depuis S3 sans intermédiaire applicatif. À ~20k vidéos et ~40 chaînes, un warehouse X-Small (1 crédit/heure) suffit largement — le coût réel est de ~$3–5/mois en compute pour un pipeline quotidien de 3 minutes.

### D2 — AWS S3 pour la couche raw

J'ai choisi AWS S3 comme object store pour la couche raw. S3 est le standard de référence pour le stockage objet cloud. Il expose la même API boto3 que tout autre store compatible S3 — seule l'URL du endpoint est une variable d'environnement. S3 et Snowflake sont tous deux sur AWS : la connexion directe via `COPY INTO` est optimisée nativement sans transfert de données supplémentaire.

### D3 — Apache Airflow avec astronomer-cosmos

J'ai utilisé Airflow pour l'orchestration avec le LocalExecutor. Pour un DAG quotidien avec ~15 tâches, le LocalExecutor exécute les tâches comme des sous-processus sur le même hôte — Redis et Celery ne sont pas nécessaires. L'intégration dbt est faite via `astronomer-cosmos` : cosmos parse le projet dbt et génère automatiquement une tâche Airflow par modèle dbt, avec les dépendances calculées depuis les `ref()`. Chaque modèle est visible individuellement dans l'UI Airflow avec ses logs et son statut.

### D4 — dbt pour les transformations SQL

J'ai utilisé dbt pour toutes les transformations SQL (staging, intermediate, core, marts). dbt apporte quatre choses qu'un SQL standalone n'a pas : un graphe de dépendances automatique calculé depuis les `{{ ref() }}` (plus besoin de câbler l'ordre dans Airflow manuellement), des matérialisations déclaratives (le DDL et l'UPSERT sont générés automatiquement selon la configuration), la documentation et le lineage auto-générés depuis les `schema.yml`, et un système de tests déclaratifs intégrés. Les modèles dbt sont des fichiers `.sql` contenant des `SELECT` — la logique SQL est identique à ce qu'on écrirait en standalone.

### D5 — dbt tests + Elementary pour la qualité des données

J'ai utilisé les dbt tests natifs pour les contrôles déclaratifs (`not_null`, `unique`, `relationships`, `accepted_range`) et Elementary pour la détection d'anomalies et la visualisation. Elementary est un package dbt qui s'installe dans `packages.yml` — il détecte statistiquement les anomalies de volume et de fraîcheur sans seuils manuels, et expose un dashboard d'historique des tests stocké dans Snowflake. Les contrôles s'exécutent comme portes bloquantes via cosmos après chaque couche : un check en échec interrompt le DAG avant propagation vers les marts.

### D6 — Apache Superset comme couche BI

J'ai connecté Superset au schema `marts` de Snowflake. Superset est un outil BI open source qui nécessite un déploiement complet : backend PostgreSQL pour les métadonnées, configuration des rôles et de l'authentification, connexion SQLAlchemy vers Snowflake. Ce déploiement est représentatif de ce qu'un DE fait en production pour mettre un outil BI à disposition des analystes. Superset s'intègre nativement avec Snowflake via le connecteur `snowflake-sqlalchemy`.

### D7 — uv pour la gestion des dépendances Python

J'ai choisi `uv` pour la gestion des dépendances. C'est un gestionnaire écrit en Rust, nettement plus rapide que pip ou poetry à l'installation, et il produit un `uv.lock` déterministe — n'importe qui qui clone le projet obtient exactement les mêmes versions. La configuration tient dans `pyproject.toml` avec deux groupes : `prod` pour le pipeline, `dev` pour les outils de développement.

### D8 — ruff et sqlfluff pour le linting et le formatage

J'utilise `ruff` pour Python (lint + format en un seul outil) et `sqlfluff` pour SQL. Les deux s'exécutent en CI à chaque push pour garantir une qualité de code cohérente.

### D9 — ELT plutôt qu'ETL

Je stocke les données brutes en premier, puis je transforme à l'intérieur du warehouse avec du SQL. Cette approche me donne deux avantages : je peux rejouer n'importe quelle date passée depuis le fichier brut sans re-solliciter l'API (et donc sans brûler de quota), et les transformations métier sont auditables et versionnées en SQL plutôt que cachées dans du code Python.

### D10 — Quatre couches physiques (raw → staging → intermediate → core → marts)

Chaque couche a une responsabilité unique et une stratégie de chargement dédiée. Une couche corrompue peut toujours être reconstruite depuis la couche en dessous, ce qui isole les pannes et simplifie le débogage.

- **raw** — fichiers JSON immuables dans S3, append-only, jamais modifiés après écriture
- **staging** — miroir fidèle de la source, toutes les colonnes en `TEXT`, matérialisé en view
- **intermediate** — casts de types et nettoyage uniquement, matérialisé en view
- **core** — star schema typé, source de vérité du warehouse, matérialisé en tables
- **marts** — tables analytiques dénormalisées pour Superset, matérialisées en tables

### D11 — Couche intermediate pour isoler les casts

La couche `intermediate` prend en charge uniquement les casts de types (`::BIGINT`, `::DATE`, `::TIMESTAMP_TZ`) et le renommage de colonnes. Le `core` ne reçoit que des colonnes déjà typées et se concentre exclusivement sur la modélisation. Si l'API change un type, l'erreur remonte dans `intermediate` — pas dans la logique métier. Chaque couche a une seule responsabilité.

### D12 — Staging et intermediate matérialisés en views

Les couches `staging` et `intermediate` sont des views dbt — elles ne stockent aucune donnée physique sur Snowflake. Ces couches reshapent de la donnée qui existe déjà dans la table source chargée par Python. Les matérialiser en tables dupliquerait le stockage sans valeur ajoutée et sans bénéfice de performance à ce volume. Seuls `core` et `marts` sont matérialisés en tables physiques, car leurs requêtes sont complexes et fréquemment consultées par Superset.

### D13 — JSON plutôt que Parquet pour la couche raw

L'API YouTube retourne du JSON nativement — je le stocke tel quel. Cela préserve une fidélité parfaite de la réponse API, évite toute interprétation de schéma à la frontière, et reste lisible à l'œil pour déboguer. Snowflake lit le JSON nativement via `COPY INTO` — une conversion en Parquet n'apporterait aucun bénéfice dans cette architecture où S3 n'est pas requêté directement avec Athena ou Spark.

### D14 — COPY INTO pour le chargement S3 → Snowflake

`COPY INTO` permet à Snowflake de se connecter directement à S3 et de charger le fichier JSON sans intermédiaire applicatif. Python déclenche uniquement la commande SQL `COPY INTO` — il ne transporte plus la donnée en mémoire. Ce pattern est natif Snowflake et élimine la dépendance à pandas pour le chargement brut.

### D15 — Star schema (Kimball) pour la couche core

J'ai modélisé le core en deux dimensions et une table de faits. Le star schema réduit le nombre de jointures par requête analytique et c'est le modèle pour lequel les outils BI sont optimisés. Le grain est sans ambiguïté : une ligne par `(vidéo, date de snapshot)` dans la table de faits, ce qui permet de conserver l'historique journalier des métriques d'engagement.

### D16 — SCD Type 1 sur toutes les dimensions

Les deux dimensions (`dim_channel`, `dim_video`) utilisent un SCD Type 1 via la matérialisation `incremental` de dbt : en cas de conflit sur la natural key, les attributs sont écrasés avec les valeurs du jour. SCD Type 2 a été évalué et écarté pour les raisons suivantes : `subscribers_count` change tous les jours — SCD Type 2 créerait une nouvelle ligne quotidiennement, transformant la dimension en table de faits déguisée. L'historique des métriques qui changent fréquemment appartient aux tables de faits, pas aux dimensions. C'est pour cela que `fct_channel_daily_snapshot` existe — elle capture l'évolution quotidienne des métriques de chaîne là où elle appartient.

### D18 — Soft delete sur `dim_video`

Quand une vidéo disparaît de l'extraction du jour (suppression par le créateur, mise en privé), je la marque `is_active = FALSE` avec `deleted_at = NOW()` plutôt que de la supprimer. Un hard delete orpheliserait toutes les lignes de faits historiques référençant cette vidéo. Si la vidéo réapparaît, elle est réactivée. La logique de soft delete est implémentée dans une macro dbt.

### D19 — Grain daily snapshot sur la table de faits

La table de faits conserve une ligne par `(video_key, snapshot_date)`. C'est le seul modèle qui permet de répondre à des questions comme "comment l'engagement a-t-il évolué dans le temps". La clé primaire composite garantit l'idempotence : rejouer la même date met à jour les métriques sans créer de doublon.

### D20 — Surrogate keys sur les dimensions

Chaque dimension a une surrogate key entière (`channel_key`, `video_key`) et conserve la natural key YouTube en colonne `UNIQUE`. Les jointures se font toujours sur la surrogate key, jamais sur la natural key. Cela isole le warehouse des changements de format côté YouTube et garantit des jointures rapides sur des entiers.

### D21 — Idempotence par conception

Ré-exécuter le pipeline sur la même date produit un état identique — aucun doublon, aucun état partiel. Les pipelines échouent et Airflow re-déclenche. Sans idempotence, chaque retry risque de créer des doublons qui nécessitent un nettoyage manuel. La matérialisation `incremental` de dbt avec `unique_key` garantit cette idempotence automatiquement.

### D22 — Contrôles qualité comme portes bloquantes

Les dbt tests s'exécutent via cosmos après chaque couche et interrompent le run en cas d'échec. Il vaut mieux un pipeline arrêté que des chiffres faux dans les dashboards. Elementary complète ces contrôles avec une détection statistique d'anomalies sans seuils manuels.

### D23 — Notifications email SMTP

J'envoie un email à chaque échec de tâche et à chaque succès du DAG. Un email d'échec remonte immédiatement le problème. Un email de succès sert de heartbeat quotidien — un email manquant est lui-même un signal d'alerte.

### D24 — Secrets dans les Variables et Connexions Airflow

Tous les secrets (clé API YouTube, credentials AWS, connexion Snowflake, SMTP) sont stockés dans les Variables et Connexions Airflow, jamais dans le code. Le DAG est versionné dans Git — un secret en dur serait exposé.

### D25 — Prometheus + Grafana pour l'observabilité du pipeline

Airflow expose ses métriques internes via StatsD (durée des tâches, taux de succès/échec, latence du scheduler). Un StatsD Exporter les convertit au format Prometheus, et Grafana les visualise en temps réel. Cela permet de détecter une dégradation progressive — par exemple une extraction de plus en plus lente signalant l'approche du quota YouTube — avant qu'elle ne devienne un échec de tâche.

### D26 — Architecture Lakehouse hybride

Le pipeline combine un object store cloud (AWS S3) pour la couche raw et un warehouse cloud (Snowflake) pour les transformations et l'analytique. Le lake garantit le stockage brut immuable à faible coût, le replay et l'audit trail. Le warehouse garantit les performances SQL analytiques et la gouvernance des données. S3 et Snowflake sont tous deux sur AWS — la connexion directe via `COPY INTO` est optimisée nativement.
