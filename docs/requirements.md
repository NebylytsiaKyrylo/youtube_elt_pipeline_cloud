# Cahier des charges — YouTube ELT Pipeline

---

## 1. Contexte du projet

Une agence marketing française spécialisée dans la tech souhaite identifier les chaînes YouTube francophones les plus influentes dans le domaine technique — Data, IA, Développement Web, DevOps, Cybersécurité — afin de conseiller ses clients sur leurs partenariats et leurs formats de contenu.

L'agence fournit une liste curatée d'environ 40 chaînes (minimum 10 000 abonnés chacune). Les données doivent être actualisées automatiquement chaque jour via l'API YouTube Data.

**Ce qui est demandé :**

- Extraire quotidiennement les métadonnées et statistiques des chaînes et vidéos depuis l'API YouTube
- Stocker les données brutes, puis les charger dans un entrepôt analytique structuré en couches
- Livrer **13 tables analytiques rafraîchies chaque jour** (marts) pour alimenter les recommandations de partenariats
- Garantir la qualité des données à chaque étape du pipeline
- Exposer les résultats dans un outil BI interactif

---

## 2. Spécifications métier

### Périmètre

L'équipe data doit livrer **13 tables analytiques rafraîchies quotidiennement** (marts) couvrant six familles de questions métier :

1. **Volume & portée** — qui domine en vues absolues et en abonnés
2. **Engagement & rétention** — qui transforme des vues en interactions
3. **Efficacité d'audience** — qui a la communauté la plus engagée relativement à sa taille
4. **Activité de publication** — qui est encore actif et depuis quand
5. **Performance par format** — quelles durées de vidéo génèrent le meilleur engagement
6. **Croissance & fiabilité** — quelles chaînes progressent et lesquelles présentent des signaux suspects

Chaque mart est reconstruit à chaque exécution via un atomic swap à partir de la couche core de l'entrepôt. Les métriques sont calculées sur le dernier snapshot journalier.

### Spécifications des marts

Chaque mart est décrit du point de vue de l'analyste data : question métier, définition des métriques, filtres, résultat attendu et contrat de colonnes.

---

#### M1 — `mart_channel_top_views`

> **Question métier :** Quelles chaînes dominent le périmètre en volume de vues absolu, sans pondération ?
> **Définition :** Somme des `video_views`, `video_likes`, `video_comments` et nombre de vidéos par chaîne, sur le dernier snapshot.
> **Filtre :** Aucun.
> **Résultat attendu :** Toutes les chaînes, triées par `total_views` décroissant.
> **Colonnes :** `channel_key`, `channel_name`, `total_videos`, `total_views`, `total_likes`, `total_comments`.

#### M2 — `mart_channel_top_subscribers`

> **Question métier :** Quelles chaînes ont la plus grande portée d'audience, indépendamment de l'engagement ?
> **Définition :** Nombre d'abonnés et agrégats vues/vidéos par chaîne.
> **Filtre :** Aucun.
> **Résultat attendu :** Toutes les chaînes, triées par `subscribers_count` décroissant.
> **Colonnes :** `channel_name`, `subscribers_count`, `total_views`, `total_videos`.

#### M3 — `mart_channel_top_likes_rate`

> **Question métier :** Quelles chaînes génèrent le taux d'approbation passive (likes par vue) le plus élevé ?
> **Définition :** `likes_rate = total_likes / total_views`.
> **Filtre :** Chaînes avec `total_views >= 10 000` pour exclure les chaînes sous-représentées en données (cohérent avec le critère d'entrée des 10 000 abonnés).
> **Résultat attendu :** Chaînes éligibles triées par `likes_rate` décroissant.
> **Colonnes :** `channel_name`, `total_views`, `total_likes`, `likes_rate`.
> **Critère d'acceptation :** `likes_rate ∈ [0, 1]`.

#### M4 — `mart_channel_size_distribution`

> **Question métier :** Comment les chaînes se répartissent-elles par tranche de vues cumulées, et comment l'engagement varie-t-il selon les tranches ?
> **Définition :** Tranches sur `total_views` : `< 1M`, `1M–10M`, `10M–50M`, `> 50M` (échelle logarithmique — les vues YouTube suivent une loi de puissance). Pour chaque tranche : nombre de chaînes, abonnés moyens, score d'engagement moyen.
> **Filtre :** Aucun.
> **Résultat attendu :** Une ligne par tranche, triée par volume croissant.
> **Colonnes :** `size_bucket`, `avg_subscribers_count`, `channel_count`, `avg_engagement_score`.
> **Critère d'acceptation :** `avg_engagement_score ∈ [0, 2]`.
> **Note :** `avg_engagement_score` est la moyenne de `(likes + comments) / views` par chaîne dans la tranche.

#### M5 — `mart_channel_subscribers_vs_engagement`

> **Question métier :** Quelles chaînes actives ont une communauté fidèle (fort engagement relatif aux abonnés) par opposition à une audience passive (beaucoup d'abonnés, peu d'interactions) ?
> **Définition :** `engagement_score = (likes + comments) / views` ; `engagement_per_subscriber = (likes + comments) / subscribers_count`.
> **Filtre :** Chaînes actives uniquement — au moins une vidéo publiée dans les 30 derniers jours. Une chaîne inactive avec un ancien pic d'engagement fausserait la comparaison.
> **Résultat attendu :** Toutes les chaînes actives, triées par `engagement_per_subscriber` décroissant.
> **Colonnes :** `channel_name`, `subscribers_count`, `total_views`, `engagement_score`, `engagement_per_subscriber`.
> **Critère d'acceptation :** `engagement_score ∈ [0, 2]`, `engagement_per_subscriber >= 0`.

#### M6 — `mart_channel_latest_videos`

> **Question métier :** Quand chaque chaîne a-t-elle publié pour la dernière fois — est-elle encore assez active pour être candidate à un partenariat ?
> **Définition :** Dernière vidéo active par chaîne ; nombre de jours écoulés depuis la publication.
> **Filtre :** `is_active = TRUE` uniquement (exclut les vidéos en soft delete).
> **Résultat attendu :** Une ligne par chaîne.
> **Colonnes :** `channel_name`, `latest_title`, `latest_published_at`, `days_since_last_video`.
> **Critère d'acceptation :** `days_since_last_video >= 0`.

#### M7 — `mart_channel_engagement_active`

> **Question métier :** Quelles chaînes actives génèrent le plus d'interactions par vue ?
> **Définition :** `engagement_score = (likes + comments) / views`. Le score est calculé sur l'intégralité du catalogue de la chaîne (pas uniquement les vidéos récentes) afin de représenter l'ensemble de la communauté. `channel_age_years` est fourni pour contexte (les chaînes anciennes accumulent davantage de vues passives).
> **Filtre :** Chaînes actives uniquement — au moins une vidéo publiée dans les 30 derniers jours.
> **Résultat attendu :** Une ligne par chaîne active, triée par `engagement_score` décroissant.
> **Colonnes :** `channel_name`, `channel_age_years`, `last_published`, `total_videos`, `total_views`, `total_likes`, `total_comments`, `engagement_score`.
> **Critère d'acceptation :** `engagement_score ∈ [0, 2]`, `channel_age_years >= 0`.

#### M8 — `mart_channel_retention`

> **Question métier :** Quelles chaînes actives ont une communauté qui interagit activement (commentaires) par opposition aux chaînes qui n'accumulent que des vues passives ?
> **Définition :** `retention_score = comments / views` (un commentaire demande un effort — signal plus fort qu'un like). `engagement_score` est fourni en complément pour comparaison.
> **Filtre :** Chaînes actives uniquement — au moins une vidéo publiée dans les 30 derniers jours.
> **Résultat attendu :** Une ligne par chaîne active, triée par `retention_score` décroissant.
> **Colonnes :** `channel_name`, `total_videos`, `total_views`, `total_comments`, `retention_score`, `engagement_score`.
> **Critère d'acceptation :** `retention_score ∈ [0, 1]`, `engagement_score ∈ [0, 2]`.

#### M9 — `mart_video_top_views`

> **Question métier :** Quelles vidéos individuelles ont le plus marqué le périmètre en nombre de vues absolu ?
> **Définition :** Top 10 vidéos par `video_views` sur le dernier snapshot, classées avec `DENSE_RANK` pour gérer les ex-æquo.
> **Filtre :** Aucun.
> **Résultat attendu :** Vidéos avec `DENSE_RANK <= 10` — peut dépasser 10 lignes en cas d'ex-æquo.
> **Colonnes :** `channel_name`, `title`, `published_at`, `video_views`, `video_likes`.

#### M10 — `mart_video_format_engagement`

> **Question métier :** Quelle durée de vidéo génère le meilleur engagement ? L'agence doit-elle recommander des contenus courts ou longs ?
> **Définition :** Tranches sur `duration_seconds / 60` : `0–3 min`, `3–7 min`, `7–15 min`, `15–30 min`, `30+ min`. Pour chaque tranche : nombre de vidéos, vues moyennes, score d'engagement moyen.
> **Filtre :** Aucun.
> **Résultat attendu :** Une ligne par tranche, triée par `avg_engagement_score` décroissant.
> **Colonnes :** `duration_bucket`, `total_videos`, `avg_views`, `avg_engagement_score`.
> **Critère d'acceptation :** `avg_engagement_score ∈ [0, 2]`.

> **Prérequis commun M11–M13 :** Ces marts requièrent que `fct_channel_daily_snapshot` conserve au moins 90 jours d'historique. La politique de rétention des snapshots doit être configurée en conséquence dans le pipeline.

#### M11 — `mart_channel_subscriber_growth`

> **Question métier :** Quelles chaînes ont la croissance d'abonnés la plus forte sur les 30 derniers jours ?
> **Définition :** `subscribers_today` = valeur au dernier snapshot. `subscribers_30d_ago` = valeur il y a 30 jours. `growth_absolute = subscribers_today - subscribers_30d_ago`. `growth_pct = growth_absolute / subscribers_30d_ago`.
> **Filtre :** Chaînes avec au moins 30 jours de données dans `fct_channel_daily_snapshot`.
> **Résultat attendu :** Toutes les chaînes éligibles, triées par `growth_pct` décroissant.
> **Colonnes :** `channel_name`, `subscribers_today`, `subscribers_30d_ago`, `growth_absolute`, `growth_pct`.
> **Critère d'acceptation :** `subscribers_today >= 0`, `growth_pct >= -1`.

#### M12 — `mart_channel_growth_anomaly`

> **Question métier :** Quelles chaînes présentent un pic de croissance suspect pouvant indiquer des abonnés achetés ?
> **Définition :** `daily_growth` = variation journalière via `LAG()`. `avg_daily_growth` = moyenne des variations journalières sur 30 jours. `spike_ratio = max_daily_growth / NULLIF(avg_daily_growth, 0)`. Une chaîne est suspecte si `spike_ratio > 5`.
> **Filtre :** Chaînes avec au moins 30 jours de données et `avg_daily_growth > 0`.
> **Résultat attendu :** Toutes les chaînes éligibles, triées par `spike_ratio` décroissant.
> **Colonnes :** `channel_name`, `max_daily_growth`, `avg_daily_growth`, `spike_ratio`, `is_suspicious`.
> **Critère d'acceptation :** `spike_ratio >= 0`, `is_suspicious IN (TRUE, FALSE)`.

#### M13 — `mart_channel_trajectory`

> **Question métier :** Sur les 90 derniers jours, quelles chaînes sont en croissance, stables ou en déclin ?
> **Définition :** `growth_pct_90d = (subscribers_today - subscribers_90d_ago) / subscribers_90d_ago`. Classification : `forte croissance` (> 20%), `croissance modérée` (5–20%), `stable` (±5%), `déclin` (< -5%).
> **Filtre :** Chaînes avec au moins 90 jours de données.
> **Résultat attendu :** Toutes les chaînes éligibles, triées par `growth_pct_90d` décroissant.
> **Colonnes :** `channel_name`, `subscribers_90d_ago`, `subscribers_today`, `growth_pct_90d`, `trajectory`.
> **Critère d'acceptation :** `trajectory IN ('forte croissance', 'croissance modérée', 'stable', 'déclin')`.

### Couche BI

Un outil BI auto-hébergé doit exposer les marts dans des tableaux de bord interactifs, sur des données réelles issues du pipeline, afin de démontrer la valeur métier de bout en bout. Les tableaux de bord sont illustratifs — les marts constituent le contrat.

