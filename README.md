# Besançon et moi

Intégration Home Assistant pour le portail **« Besançon et moi »**
(besancon-emoi.fr) de la Ville de Besançon et de Grand Besançon Métropole.

Le portail n'expose pas d'API publique : cette intégration s'appuie sur les
pages du site (connexion, contrats d'eau, factures, historique d'index,
saisie du relevé de compteur). Elle est **indépendante de
[tarifeau](https://github.com/Mathilde-Team/tarifeau)** : les deux se
complètent via les entités Home Assistant (tarifeau peut calculer une
facture à partir de l'index publié ici).

## Fonctionnalités

- Connexion à votre espace « Besançon et moi » (e-mail + mot de passe).
- Par contrat d'eau :
  - **Index du compteur** (m³), avec date et type de relevé, et historique ;
  - **Consommation moyenne journalière** (m³/j) ;
  - **Dernier prélèvement** (€) : montant prélevé par le portail
    (mensualité ou ajustement annuel — ce n'est pas le total de la facture),
    avec date, n° de facture et lien du PDF en attributs ;
  - **Dernière facture (TTC)** (€) : total TTC lu dans le PDF de la
    dernière facture (le portail ne le publie pas), avec date,
    n° de facture et montant prélevé en attributs. Le PDF est téléchargé
    une seule fois par facture ; s'il est inaccessible ou illisible, le
    capteur reste inconnu et la mise à jour continue.
- Par contrat déchets (bac roulant) :
  - **Dernière pesée** et **Poids de la dernière pesée** (kg), avec type de
    bac, volume, nombre de vidages, n° de bac et n° de puce en attributs ;
  - **Dernière pesée bac jaune** et **Poids de la dernière pesée bac jaune**
    (kg) pour les bacs recyclables ;
  - la liste des **20 dernières pesées** en attribut de chaque capteur ;
  - la table du portail est paginée : toutes les pages sont lues.
- Calendrier de collecte des déchets (open data Grand Besançon, sans
  identifiants) :
  - **Collecte résiduels** et **Collecte recyclables** (jour + parité
    `toutes les semaines` / `semaines paires` / `semaines impaires`), avec
    prochaine date, compte à rebours en jours, adresse et calendrier PDF
    officiel en attributs ;
  - une entité **calendrier** au format Home Assistant (aperçu du calendrier,
    automations sur les événements) ;
  - configuration par commune et rue du formulaire
    [open data](https://data.grandbesancon.fr/opendata/dataset/joursDeCollecteDechets/)
    (68 communes), dans les options de l'intégration ;
  - les erreurs open data n'interrompent pas les autres entités.
- Action `besancon_emoi.saisir_index` pour transmettre un relevé de compteur,
  avec ciblage par contrat quand plusieurs compteurs sont configurés.
- **Saisie hebdomadaire automatique** : l'index est lu sur une entité de
  votre choix (par exemple le compteur wM-Bus d'
  [Esphome-Compteur-wmbus](https://github.com/Mathilde-Team/Esphome-Compteur-wmbus))
  et transmis au portail une fois par semaine, le jour choisi.

## Installation

### HACS (recommandé)

1. Ajoutez ce dépôt dans HACS (« Dépôts personnalisés », catégorie
   « Intégration »).
2. Recherchez « Besançon et moi » et installez-la.
3. Redémarrez Home Assistant.

### Manuel

Copiez `custom_components/besancon_emoi` dans votre dossier `config/custom_components/`.

## Configuration

1. Réglages → Appareils et services → Ajouter une intégration →
   « Besançon et moi ».
2. Saisissez l'e-mail et le mot de passe de votre compte besancon-emoi.fr.

### Options

- **Saisie hebdomadaire automatique** : active la transmission automatique.
- **Jour de la semaine** : jour d'envoi du relevé.
- **Entité source de l'index** : `entity_id` du capteur qui publie l'index
  absolu du compteur (en m³).
- **Entité source par contrat** (comptes à plusieurs contrats d'eau) : un
  champ « index_entity::<contrat> » par contrat connu ; la saisie
  hebdomadaire transmet à chaque compteur l'index de sa propre entité.
  Sans appariement, l'automatisme reste limité au compte à contrat unique
  (jamais le même index envoyé à plusieurs compteurs).

### Exemple d'automatisation (saisie manuelle hebdomadaire)

```yaml
automation:
  - alias: Relevé hebdomadaire du compteur d'eau
    triggers:
      - trigger: time
        at: "08:00:00"
    conditions:
      - condition: time
        weekday: mon
    actions:
      - action: besancon_emoi.saisir_index
        data:
          index: "{{ states('sensor.compteur_eau_index') | int }}"
          contrat: "123456-02"  # obligatoire si plusieurs contrats
```

## Tableau de bord

Une vue prête à l'emploi est fournie dans
[`exemples/tableau-de-bord.yaml`](exemples/tableau-de-bord.yaml) :

- **prochaines collectes** : jour et échéance (aujourd'hui, demain, dans
  N jours) des résiduels et des recyclables, adresse et lien vers le
  calendrier PDF ;
- **mes contrats** : un tableau par contrat (eau, déchets) avec toutes ses
  mesures, sans rien à adapter ;
- **calendrier de collecte** et **consommation d'eau sur 30 jours** :
  remplacer les entités d'exemple par les vôtres.

Seules des cartes intégrées à Home Assistant sont utilisées. Installation :
Paramètres → Tableaux de bord → ouvrir un tableau de bord → ⋮ → Modifier
→ ⋮ → Éditeur de configuration, puis coller le contenu du fichier comme
nouvel élément de `views:`.

## Diagnostic

Pour suivre pas à pas la connexion au portail, la lecture des contrats et la saisie du relevé, activer le journal de debug dans `configuration.yaml` :

```yaml
logger:
  logs:
    custom_components.besancon_emoi: debug
```

Le courriel, le mot de passe et la session ne sont jamais écrits ; les autres paramètres sont résumés (tailles, clés, types, début des pages tronqué à 60 caractères).

## Limites

- Le portail est un site web : toute évolution de son HTML peut casser
  l'intégration (les tests capturent la structure observée).
- La facture reste disponible en PDF sur le portail ; l'intégration publie
  le montant prélevé et les dates, pas le document (le lien relatif du PDF
  est fourni en attribut du capteur).
- La saisie hebdomadaire automatique prend en charge plusieurs contrats
  d'eau via l'appariement explicite « contrat → entité source » dans les
  options (un champ par contrat) ; sans appariement, elle ne cible qu'un
  compte à contrat unique (jamais le même index envoyé à plusieurs
  compteurs) et l'action manuelle exige le champ « contrat ».
- L'entité source d'index doit publier l'index **absolu** du compteur en m³
  (pas une consommation périodique ni des litres).
- Le mot de passe du portail est stocké par Home Assistant comme pour toute
  intégration à identifiants ; la session est dédiée à chaque compte.
- Aucune donnée n'est envoyée ailleurs qu'au portail officiel.

## Données et licences

- **Portail « Besançon et moi »** : ce n'est pas une source de données
  ouvertes. L'accès se fait avec le compte personnel de l'usager, sous les
  [conditions d'utilisation du service](https://www.besancon-emoi.fr/account/accept)
  (CGU) de la Ville de Besançon et de Grand Besançon Métropole. Les CGU ne
  parlent pas de l'accès automatisé. Les clauses qui s'appliquent ici sont
  les suivantes :
  - art. 11 : l'usager n'a sur les informations du service qu'un **droit
    d'usage privé, non collectif et non commercial**. Il ne doit ni entraver
    ni perturber le service, ni collecter des données d'autres usagers ;
  - art. 18 : la reproduction n'est autorisée que pour un usage **strictement
    personnel et interne**, et toute extraction d'une partie substantielle
    des informations est interdite. Le droit d'utiliser les logiciels du
    service est personnel et non cessible ;
  - art. 10 : l'usager est seul responsable de la confidentialité de ses
    identifiants ;
  - art. 19 : le service est fourni « en l'état », et ses informations n'ont
    qu'une **valeur indicative**.

  L'intégration est conçue pour rester dans ce cadre. Chaque usager la
  configure avec son propre compte et ne lit que ses propres données :
  contrats, factures, index. Les données restent dans son Home Assistant et
  ne sont transmises à aucun tiers. Le portail est interrogé toutes les
  6 heures, soit une session par compte. Aucune donnée n'est rediffusée.
  L'usager reste responsable de l'usage qu'il fait de son compte. En cas de
  doute sur l'usage d'un logiciel tiers, la Ville peut être contactée
  (besancon-emoi@besancon.fr).
- **Jours de collecte des déchets** : ils viennent du jeu de données ouvert
  [Jours de collecte des déchets](https://data.grandbesancon.fr/opendata/dataset/joursDeCollecteDechets/)
  d'OpenData Besançon, publié sous **Licence Ouverte** (Etalab) : réutilisation
  libre en citant la source, « Grand Besançon Métropole — OpenData Besançon ».
- **Dépendances** (manifest) : beautifulsoup4 (MIT), pypdf (BSD-3-Clause),
  d'après PyPI.
- **Code** : licence [MIT](LICENSE) (intégration Home Assistant, exception
  SOCLE §1).

## Développement

```bash
uv sync
uv run --only-group lint ruff check .
uv run -p 3.13 --only-group test pytest --cov
```

### Essayer hors de Home Assistant

Le client du portail se teste sans Home Assistant (lecture seule :
connexion, contrats, historique d'index, derniers prélèvements — aucune
saisie de relevé) :

```bash
BESANCON_EMOI_EMAIL=... BESANCON_EMOI_PASSWORD=... \
    uv run -p 3.13 --only-group test python scripts/tester_hors_ha.py
```

Sans variables d'environnement, les identifiants sont demandés au
lancement. La journalisation de l'authentification reste bavarde (niveau
INFO) tant qu'aucune connexion n'a réussi, puis redevient silencieuse
(DEBUG) — et repasse en mode bavard au premier échec.

## Socle qualité Mathilde-Team

Ce dépôt suit le [socle qualité de l'équipe](https://github.com/Mathilde-Team/.github/blob/main/SOCLE.md) :
branches protégées `dev` (intégration) et `main` (livraison), CI requise
(ruff, yamllint, pytest, hassfest, validation HACS), licence MIT.
