# FEATURE 02 — Passage contrôlé en production et gestion des capitaux par exchange

**Phase :** 2 — conditionnelle (ne pas implémenter pendant la phase 1)  
**Statut :** Spécification d'évolution, activation soumise à décision explicite  
**Prérequis :** FEATURE 01 opérationnelle et performances paper trading jugées suffisamment robustes, vérification réglementaire et disponibilité des exchanges pour le compte concerné.  
**Référence technique :** `/AGENTS.md`.

## 1. Objectif métier

Après validation de l'arbitrage en paper trading, permettre le **Spot inter-exchanges avec capitaux réels**, par exemple **100 EUR initialement par exchange**, puis l'augmentation indépendante et progressive des fonds disponibles sur chaque plateforme.

La phase 2 doit réutiliser le calcul d'opportunité, l'API REST et les contrats métier de la phase 1, **sans réécrire le moteur**. Seule la source des soldes, le mode d'exécution, la réconciliation et les contrôles de sécurité changent.

## 2. Décisions de périmètre

- Activer d'abord **deux exchanges** pour lesquels le compte réel, les API Spot, les paires, la réglementation MiCA, les frais, les limites d'ordre et les autorisations sont effectivement validés; ajouter les autres par adaptateurs.
- Trading **Spot uniquement**; aucun levier, future, margin, short ou financement emprunté.
- Dépôts effectués par l'utilisateur **directement sur les plateformes**, dans leurs circuits réglementaires et sécurisés; notre application **observe et réconcilie** les fonds reçus, mais ne prétend jamais initier un virement bancaire grâce à un simple endpoint interne.
- **Pas de fonction de retrait** depuis le moteur d'arbitrage; les clés API doivent être limitées au minimum (lecture et trading Spot si autorisé, sans permission de retrait).
- Modes `Paper` et `Live` strictement isolés, avec identités, secrets, portefeuilles et stockage d'exécution différenciés. Passage en live via procédure d'exploitation autorisée, jamais simple POST public.

## 3. Scénarios fonctionnels et critères d'acceptation

### F02-S01 — Enregistrer et connecter un compte exchange réel
**Étant donné** un exchange approuvé pour le territoire, **quand** un opérateur configure les identifiants API dans des Kubernetes Secrets, **alors** le service teste les droits et la connectivité en lecture seule avant de rendre la venue disponible en trading.

**Acceptation :** secrets jamais dans Git, les journaux ou une réponse API; vérification des permissions réelles; lecture seule possible sans trading; aucun droit de retrait nécessaire; erreur explicite si la paire n'est pas accessible.

### F02-S02 — Activer la production sans ambiguïté
**Étant donné** une installation configurée en Paper, **quand** la production est explicitement approuvée, **alors** une configuration/déploiement séparé habilite le moteur Live après vérification des prérequis.

**Acceptation :** refus par défaut; absence de secrets live en Paper; contrôles de permission, ordre et solde testés avant activation; journalisation de la décision et kill switch opérationnel; aucun endpoint non authentifié permettant d'activer Live.

### F02-S03 — Lire et réconcilier les capitaux réels
**Étant donné** un compte Spot connecté, **quand** l'application démarre ou un solde change, **alors** elle interroge les soldes réels et rapproche les changements avec les opérations connues.

**Acceptation :** les soldes de l'exchange font autorité; différences enregistrées et signalées; ordres en cours et capital bloqué distingués; redémarrage sans duplication des dépôts/fills; dates et identifiants sources conservés.

### F02-S04 — Ajouter du capital à une plateforme
**Étant donné** 100 EUR sur OKX et 100 EUR sur Kraken, **quand** l'utilisateur dépose 200 EUR supplémentaires sur OKX via son compte exchange, **alors** le système détecte ou réconcilie le dépôt et rend le capital utilisable **sur OKX uniquement** après confirmation.

**Acceptation :** bilan OKX augmente, Kraken inchangé; événement `Deposit` unique et auditable; capital apporté séparé du profit; aucune création artificielle de solde depuis l'API; délai et statut de confirmation visibles.

### F02-S05 — Piloter les allocations et limites
**Étant donné** des montants différents par exchange, **quand** l'opérateur définit un montant maximum par ordre, une réserve minimale ou un plafond d'exposition, **alors** le moteur respecte ces contraintes sans prétendre que les fonds sont interchangeables entre exchanges.

**Acceptation :** limites par exchange et par actif, limites globales, plafonds de pertes, arrêt d'urgence; rejet documenté avant toute soumission; interdiction de positions négatives ou de vente à découvert.

### F02-S06 — Exécuter un arbitrage réel à deux jambes
**Étant donné** deux carnets compatibles et des soldes confirmés, **quand** une opportunité rentable après frais est détectée, **alors** le moteur propose ou place, selon configuration approuvée, les deux ordres avec identifiants d'idempotence propres à chaque exchange et suit leur statut réel.

**Acceptation :** contrôle de fraîcheur des prix, tailles d'ordre, balances libres, prix limites et timeouts; état explicite si un seul ordre est exécuté; compensation ou arrêt du trading selon politique déterministe; aucune promesse d'exécution atomique; rapprochement avec le statut exchange.

### F02-S07 — Reprise sur incident et expositions résiduelles
**Étant donné** une panne réseau ou un redémarrage pendant les ordres, **quand** le service reprend, **alors** il réconcilie ordres et fills existants **avant toute nouvelle décision**.

**Acceptation :** aucun double ordre après retry; capacité à figer la stratégie; alertes si état incertain; protection contre carnet périmé et déconnexion; procédure de récupération documentée.

### F02-S08 — Rééquilibrer les portefeuilles
**Étant donné** des exécutions qui déplacent les inventaires BTC et EUR entre exchanges, **quand** les réserves ne permettent plus d'arbitrages, **alors** le système identifie le besoin de rééquilibrage et chiffre les coûts/contraintes.

**Acceptation :** d'abord recommandation et approbation manuelle; aucun transfert inter-exchanges automatique au lancement; frais de réseau, dépôt/retrait, devise et délais pris en compte dans les résultats; les transferts ne comptent pas comme profit.

### F02-S09 — Suivre capital, bénéfices et disponibilité via l'API
**Étant donné** des comptes alimentés à des dates différentes, **quand** on consulte l'API, **alors** elle présente capital versé par exchange, valeur totale, liquidité disponible, inventaire, transferts, résultat net et PnL séparé des variations de marché.

**Acceptation :** API compatible MAUI / Konektag; journal auditable; droits d'accès par rôle; aucune fuite de clés ou de données secrètes.

## 4. Parcours métier : ajout progressif de capital

Exemple (valeurs illustratives) :

| Moment | OKX | Kraken | Bitpanda | Capital cumulé apporté |
| --- | ---: | ---: | ---: | ---: |
| Début (réel) | 100 € | 100 € | 100 € | 300 € |
| Dépôt ultérieur OKX | 300 € | 100 € | 100 € | 500 € |
| Dépôt ultérieur Kraken | 300 € | 200 € | 100 € | 600 € |

Ces colonnes représentent des **apports cumulés**, pas nécessairement les soldes disponibles après trading ou variations de prix.

À chaque dépôt réel :
1. Le client dépose sur le site/application de l'exchange.
2. L'exchange constate le dépôt.
3. Notre service lit le solde/les mouvements accessibles via son API.
4. Un rapprochement valide l'origine et le statut du mouvement.
5. Le ledger attribue le nouvel apport à cet exchange et à cet actif.
6. Le moteur peut utiliser la partie confirmée et libre dans la limite des règles de risque.

Ne jamais considérer le capital envoyé comme disponible avant confirmation par l'exchange.

## 5. Contrats API prévus (exemples, soumis à sécurisation)

Garder les endpoints phase 1. Ajouter progressivement :
- `GET /api/v1/live/accounts` : comptes configurés, état et permissions, **sans secrets**.
- `GET /api/v1/live/balances` : fonds disponibles, bloqués, positions Spot par venue.
- `GET /api/v1/live/deposits` : dépôts détectés et rapprochés.
- `GET /api/v1/live/orders` : ordres, états et exécutions.
- `GET /api/v1/live/reconciliation` : écarts et incidents.
- `GET /api/v1/live/risk` : limites et état du kill switch.

Toute mutation de configuration de risque exige contrôle d'accès fort et audit. **Pas de `POST /deposits` simulant un dépôt réel, pas d'API de retrait, pas d'endpoint public d'activation de Live.**

## 6. Sécurité, conformité et exploitation

- Validation des autorisations MiCA et disponibilités produit/paires spécifiques à la juridiction et au type de compte avant lancement.
- Clés d'exchange dans Kubernetes Secrets avec droits minimaux, rotation documentée et exclusion des logs.
- Authentification et autorisation de toutes les fonctions sensibles de l'API; exposition réseau privée par défaut.
- OpenTelemetry pour latence des ordres, échecs/retries, désynchronisation des soldes, exposition résiduelle et kill switch, raccordé aux composants déjà présents de K3s.
- Séparation forte des schémas/bases ou des environnements Paper et Live; jamais de portefeuille fictif utilisé comme solde réel.
- Migrations PostgreSQL versionnées et réversibles lorsque possible.
- Manifests/scripts de déploiement et de rollback **entièrement rejouables**.
- Préparation d'un mode de reprise sans trading et validation manuelle des incidents incertains.

## 7. Tests d'acceptation avant le moindre ordre réel

- Paper ne peut jamais invoquer un endpoint d'ordre réel, même avec une erreur de configuration.
- Démarrage Live sans approbation/credentials/contrôles OK : refus.
- Dépôt réel vu deux fois : un seul événement comptable et aucune augmentation fictive du solde.
- Ajout de fonds sur un exchange : aucun impact sur les comptes des autres.
- Achat exécuté, vente échouée : exposition résiduelle enregistrée et circuit de protection déclenché.
- Redémarrage après timeout : récupérer l'état réel de l'ordre sans le dupliquer.
- Pertes/plafonds franchis : kill switch et arrêt des nouvelles transactions.
- Clés API sans droit de retrait; logs et API exempts de secrets.
- Réconciliation indépendante des soldes et ordre des événements; reporting distinguant apports et profits.
- Répéter les scénarios d'incident sur Testnet/Demo **si un environnement représentatif est accessible**, sinon avec fixtures et adaptateurs simulés; passage en petit réel uniquement sur validation explicite.

## 8. Ordre de livraison

1. Vérifier comptes, droits, réglementation, disponibilité produits et fournisseurs d'API; documenter les contraintes confirmées.
2. Adapter seulement deux exchanges à la lecture **authentifiée** et à la réconciliation des soldes.
3. Ajouter le suivi des dépôts externes, comptes/permissions et limites d'exposition.
4. Intégrer l'exécution réelle derrière le contrat existant, tester les échecs et la reprise.
5. Activer sous contrôle opérationnel avec ordres très limités et surveillance renforcée.
6. Ajouter ensuite Bitpanda / autres exchanges et les outils de rééquilibrage si l'économie le justifie.

## 9. Hors périmètre initial phase 2

- Achat/vente directionnels pilotés par LLM.
- Futures, levier, marge, lending ou short.
- Transferts automatiques d'argent ou de cryptomonnaies.
- Gestion bancaire intégrée ou service de dépôt/custody.
- Nouvelle plateforme de monitoring, nouveau serveur PostgreSQL, nouvelle interface obligatoire.
- Généralisation multi-utilisateur / SaaS tant que le cas d'usage n'est pas validé.

**Critère de réussite :** après approbation explicite, la même API et le même moteur arbitrent sur comptes Spot réels de façon contrôlée, savent détecter/réconcilier des apports progressifs par venue, calculent la rentabilité nette réelle et maintiennent une piste d'audit et un mécanisme d'arrêt fiable.
