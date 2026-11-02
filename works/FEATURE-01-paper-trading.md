# FEATURE 01 — Arbitrage Spot inter-exchanges en paper trading

**Phase :** 1 — MVP obligatoire  
**Statut :** À implémenter  
**Références techniques :** `/AGENTS.md` (invariants techniques, sécurité, infrastructure)  
**Périmètre :** simulation avec données publiques, aucun fonds réel et aucun ordre envoyé à un exchange.

## 1. Objectif métier

Déterminer, sur des conditions de marché observées en temps réel, si l'achat d'un actif Spot sur un exchange et sa vente quasi simultanée sur un autre peuvent produire un **profit net réellement plausible**, avec **100 EUR de capital virtuel par exchange**. Le MVP doit permettre d'invalider rapidement l'hypothèse, pas d'afficher artificiellement des gains.

## 2. Scénario utilisateur principal

**Étant donné** un système configuré avec deux exchanges disposant chacun d'un portefeuille virtuel indépendant, **lorsqu'un** écart exploitable apparaît entre leurs carnets d'ordres, **alors** le système calcule la rentabilité *après tous les coûts*, valide les soldes, simule l'achat et la vente, enregistre les deux jambes et actualise les portefeuilles. Si les conditions ne sont pas réunies, il **refuse** l'opération et conserve la raison du refus.

Aucun utilisateur ne doit avoir à choisir manuellement les trades. Les données, décisions et résultats doivent être accessibles via l'API REST, pour une interface MAUI ou Konektag ultérieure.

## 3. Périmètre initial et hypothèses

- **2 exchanges en premier**, sélectionnés parmi OKX Europe, Kraken et Bitpanda Fusion selon disponibilité vérifiée des **carnets Spot publics**; ajouter le troisième seulement après la validation bout en bout. Ne jamais inventer une API publique Bitpanda ou des paires disponibles.
- **1 paire Spot commune en premier** (préférence BTC/EUR, si effectivement disponible et suffisamment liquide sur les deux venues). Élargir ensuite, si les mesures le justifient.
- Capital virtuel de départ **100 EUR équivalent par exchange**. Initialiser par défaut 50 EUR de devise de cotation et l'équivalent de 50 EUR en actif de base sur chaque exchange, avec valorisation initiale enregistrée et paramètres configurables.
- Modes **Paper** et **Replay** uniquement. Pas de compte utilisateur sur les exchanges, pas de clé privée, pas de Testnet obligatoire, pas de dépôt réel.
- Les opportunités sont mesurées en continu à partir des données publiques (WebSocket si disponible, REST borné en fallback).
- Utiliser la devise réelle de chaque marché, sans assimiler EUR, USD, USDC et USDT; conversion explicite si indispensable. Aucun short, levier ou financement emprunté.

## 4. Scénarios fonctionnels et critères d'acceptation

### F01-S01 — Configuration des exchanges et des marchés
**Étant donné** la configuration d'une liste de venues et de paires, **quand** le service démarre, **alors** il vérifie la disponibilité des marchés et démarre uniquement les adaptateurs compatibles. Un exchange non disponible ne doit pas bloquer la surveillance des autres.

**Acceptation :** statut de connexion visible par API; erreurs explicites; aucun endpoint inventé; configuration externe sans recompilation.

### F01-S02 — Collecte de carnets de prix comparables
**Étant donné** un marché présent sur deux exchanges, **quand** les carnets arrivent, **alors** le moteur conserve bid/ask, profondeurs utiles, tick/step size et timestamps exchange + réception locale.

**Acceptation :** refus si carnet incomplet, incohérent ou trop ancien; horodatages en UTC; pas de comparaison des derniers prix de transaction; surveillance des retards et rate limits.

### F01-S03 — Détection d'opportunités nettes
**Étant donné** les carnets de A et B, **quand** `bid(B) > ask(A)`, **alors** le moteur évalue le coût d'acheter sur A et le produit de vendre sur B, et inverse également le sens A/B.

Calcul sur une quantité effectivement échangeable et sur les **niveaux de profondeur**, avec prix moyen pondéré, commissions appliquées à chaque jambe, arrondis, minimum notionnel, slippage/latence paramétrés, éventuels frais de conversion et **provision pour rééquilibrage**.

**Acceptation :** une opportunité n'est éligible que si le profit estimé après coûts dépasse un seuil configurable; résultat contient le détail chiffré et les raisons de rejet. Ne jamais confondre spread affiché et spread exécutable.

### F01-S04 — Exécution fictive à deux jambes
**Étant donné** une opportunité acceptée et les soldes virtuels disponibles, **quand** la simulation s'exécute, **alors** elle réserve le capital nécessaire sur chaque exchange, simule les deux jambes, déduit les frais et crédite/débite les actifs sur les **bons** portefeuilles.

**Acceptation :** aucun solde disponible négatif; pas de double consommation du capital; prise en compte de la profondeur et d'une latence simulée; état distinct pour absence/partiel d'exécution et risque résiduel d'une seule jambe; pas d'hypothèse d'atomicité entre exchanges; identifiants et événements persistés; protection contre les doublons après redémarrage.

### F01-S05 — Portefeuille virtuel et apports additionnels
**Étant donné** 100 EUR virtuels sur chaque venue, **quand** l'utilisateur ajoute par API 50 EUR virtuels sur un exchange donné, **alors** seul le portefeuille de cet exchange augmente, avec un événement `Deposit` auditable.

**Acceptation :** `POST /api/v1/paper/deposits` protégé par authentification, incluant exchange, asset, amount, reference et idempotency key; montants > 0; une même clé ne peut créditer deux fois; historique immuable; les apports ne sont jamais comptés comme profit. Actifs et montant configurables; pas de transfert d'argent réel par l'API.

### F01-S06 — Suivi du résultat économique
**Étant donné** des transactions fictives et un inventaire en BTC/monnaie de cotation, **quand** on consulte l'API, **alors** on voit les soldes et valeurs par exchange et actif, le PnL d'arbitrage net, les frais et provisions de rééquilibrage, la variation de valeur des actifs détenus, le capital externe apporté et les limites de risque.

**Acceptation :** distinguer PnL réalisé, mark-to-market et apports; résultats par venue et consolidés; aucune rentabilité positive affichée en raison d'un simple dépôt; benchmark explicite et métriques non interprétées abusivement.

### F01-S07 — Replay reproductible
**Étant donné** une capture des événements de carnets et des paramètres de simulation, **quand** on rejoue la séquence avec une horloge virtuelle, **alors** les décisions, exécutions fictives, soldes et résultats sont identiques.

**Acceptation :** replay hors réseau et sans requête aux exchanges; version des paramètres et des règles conservée; tests automatisés de bout en bout; un redémarrage ne rejoue pas deux fois la même transaction.

### F01-S08 — API, exploitation et observabilité
**Étant donné** un déploiement K3s, **quand** le worker et l'API fonctionnent, **alors** l'API expose au minimum les exchanges, opportunités, balances, trades, performance, journal et dépôts virtuels.

**Acceptation :** OpenAPI; health checks; télémétrie OpenTelemetry vers le pipeline existant Alloy/Prometheus/Grafana/Loki suivant configuration réelle; PostgreSQL partagé et Kubernetes Secrets; pas de nouveau PostgreSQL, monitoring ni interface graphique; déploiement rejouable.

## 5. Règles métier non négociables

1. Les comptes virtuels sont isolés **par exchange et par actif**. Un achat sur A n'utilise jamais les fonds de B.
2. Un arbitrage nécessite simultanément la devise d'achat sur A et l'actif de vente sur B, préfinancés virtuellement.
3. Les portefeuilles se déséquilibrent après des arbitrages; les coûts et les besoins de rééquilibrage doivent être mesurés même si le MVP ne rééquilibre pas automatiquement.
4. Un résultat d'arbitrage positif **avant frais** n'autorise pas un trade.
5. Le cash apporté, les transferts et l'appréciation de l'inventaire crypto ne doivent pas être mélangés au profit d'arbitrage.
6. Interdire tout ordre réel ou l'utilisation de credentials live pendant cette phase.
7. Tout calcul financier est en decimal avec conversions et arrondis définis.

## 6. API minimale visée

- `GET /api/v1/exchanges`
- `GET /api/v1/opportunities`
- `GET /api/v1/portfolios`
- `GET /api/v1/trades`
- `GET /api/v1/performance`
- `GET /api/v1/paper/ledger`
- `POST /api/v1/paper/deposits`
- `GET /health/live` et `GET /health/ready`

Les endpoints de mutation doivent être sécurisés. Implémenter les endpoints au rythme des vertical slices, pas tous artificiellement dès le premier commit.

## 7. Tests d'acceptation prioritaires

- Prix favorables mais frais supérieurs au spread : **aucun ordre**.
- Spread favorable mais profondeur insuffisante ou carnet périmé : **aucun ordre**.
- 100 EUR sur A et B, avec BTC seulement sur B : achat A/vente B possibles sous contraintes de volume.
- Solde insuffisant / quantité minimale invalide : rejet documenté, sans débit.
- Deux opportunités concurrentes : un même euro ne peut être dépensé deux fois.
- Achat exécuté virtuellement, vente partielle/non exécutée : exposition et résultat correctement enregistrés.
- Dépôt virtuel de 100 EUR sur A, répétition avec même clé : un seul crédit; ROI et PnL inchangés par l'apport.
- Capture et replay : mêmes trades, soldes, frais et performances.
- Retrait d'un exchange de la configuration : les autres continuent sans changement du moteur.

## 8. Découpage d'implémentation (Codex)

**Slice A — Observation :** solution `net11.0` + API + PostgreSQL existant + K3s + télémétrie; 2 venues publiques, 1 paire, capture/replay, affichage des écarts et rejets.

**Slice B — Paper trading :** soldes virtuels, simulation à deux jambes, gestion coûts/risques, journal et `/paper/deposits`; tests déterministes.

**Slice C — Évaluation :** reporting net, inventaires, rebalancing estimé, fonctionnement continu, troisième exchange seulement si données prouvées.

**Critère de sortie de phase :** une campagne paper trading exploitable avec données réelles publiques, résultats nets traçables et rejouables, sans aucun ordre réel. Une semaine valide la chaîne technique, pas la rentabilité statistique.

**Hors périmètre :** LLM, TradingAgents, Freqtrade, génération de stratégies, mobile/UI, transferts réels, retrait de fonds, bots de trading directionnel, microservices, optimisation complexe.
