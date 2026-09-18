# Spécification finale — Suivi des KPI de prospection

Portail `27023376` — IÉSEG CONSEIL Paris. Version du 18 septembre 2026.

Document consolidé à partir de la note de cadrage, de `audit-kpi-prospection.md`,
de `plan-implementation.md` et de l'export `deals-developpement-commercial-a-qualifier.csv`.

**Aucune modification n'a été effectuée dans HubSpot.** Ce document est une spécification
à exécuter, pas un compte rendu d'exécution.

Légende d'exécution utilisée dans tout le document :

| Marque | Signification |
| --- | --- |
| **API** | Automatisable via l'API HubSpot |
| **MCP** | Vérifiable avec les outils MCP de cette conversation (lecture seule) |
| **MANUEL** | Obligatoirement à la main dans l'interface HubSpot |

---

## 1. Propriétés à créer

Les quatre noms internes ci-dessous ont été vérifiés comme **libres** dans le portail.
Aucune propriété à créer côté Contact : le canal y existe déjà (§2.1).

### P-01 · Canal de prospection

| Champ | Valeur |
| --- | --- |
| Objet | **Deal** (Transaction) |
| Nom interne | `canal_prospection` |
| Libellé | Canal de prospection |
| Type | Liste déroulante — énumération, sélection unique |
| Groupe | Informations sur la transaction |

Options — **valeurs internes strictement identiques** à celles de `provenance` (Contact),
condition d'une recopie sans perte par le workflow W-01 :

`Cold Calling` · `Campagne mail` · `Mailing personnalisé` · `Linkedin` · `Événement` ·
`Écosystème` · `Réseau personnel`

**Objectif** — porter au niveau de la transaction le canal par lequel le contact à l'origine
de l'affaire a été touché. C'est le chaînon manquant de l'exigence F-05 : le canal existe
aujourd'hui côté Contact mais n'est jamais rattaché au chiffre d'affaires, qui se mesure
au niveau Deal.

### P-02 · Famille de canal

| Champ | Valeur |
| --- | --- |
| Objet | **Deal** |
| Nom interne | `famille_canal` |
| Libellé | Famille de canal |
| Type | Liste déroulante — énumération, sélection unique |
| Options | `Calling` · `Mailing` · `Autre` |

**Objectif** — le CDC demande des tableaux de bord Calling vs Mailing, pas une ventilation
en sept canaux. Cette propriété fournit l'axe de filtrage direct et évite de reconstruire un
regroupement dans chacun des rapports. Alimentée automatiquement par W-02.

### P-03 · Motif de CE non présentée

| Champ | Valeur |
| --- | --- |
| Objet | **Deal** |
| Nom interne | `motif_ce_non_presentee` |
| Libellé | Motif de CE non présentée |
| Type | Liste déroulante — énumération, sélection unique |

Options : `No show client` · `Reporté à l'initiative du client` · `Reporté à notre initiative` ·
`Annulée — projet abandonné` · `Annulée — budget` · `Contact injoignable` · `Autre`

**Objectif** — rendre le KPI No show mesurable. La propriété actuelle
`raison_ce_non_presentee` est en **texte libre** et renseignée sur 4 transactions sur 465 :
ni agrégeable, ni filtrable.

Le nom retenu s'aligne sur la famille déjà en place dans le portail — `motif_de_perte`,
`motif_d_abandon`, `motif_de_rupture` — toutes en liste déroulante. Vérification faite :
aucune de ces trois ne couvre le No show, la nouvelle propriété ne fait donc pas doublon.

### P-04 · Fiabilité du canal *(recommandée)*

| Champ | Valeur |
| --- | --- |
| Objet | **Deal** |
| Nom interne | `canal_fiabilite` |
| Libellé | Fiabilité du canal |
| Type | Liste déroulante — énumération, sélection unique |
| Options | `Preuve CRM` · `Déclaratif` · `Non qualifié` |

**Objectif** — tracer *comment* le canal a été déterminé. L'export des 100 deals historiques
montre que **seuls 30 portent une activité tracée** : les 70 autres ne pourront être qualifiés
que de mémoire, ou pas du tout. Sans cette propriété, une reprise déclarative devient
indiscernable d'une donnée mesurée, et les taux de transformation par canal deviennent
ininterprétables.

Propriété optionnelle sur le plan fonctionnel, mais c'est la seule garantie que les KPI
resteront lisibles dans deux ans. Si elle est écartée, la reprise historique (§3) doit alors
se limiter aux 30 deals sourcés.

---

## 2. Propriétés existantes

### 2.1. À conserver et réutiliser telles quelles

| Propriété | Objet | Rôle | État |
| --- | --- | --- | --- |
| `provenance` | Contact | **Source de vérité du canal.** Ne pas dupliquer, ne pas recréer. | 7 options correctes, mais renseignée 1 fois / 13 751 |
| `date_de_presentation_ce` | Deal | Base de calcul du No show | 244 / 465 — exploitable |
| `dealstage` « CE non-présentée » (`1862597857`) | Deal | Étape du No show | Existe déjà, **rien à créer** |
| `amount_in_home_currency` | Deal | Base de tous les rapports CA | Standard HubSpot |

### 2.2. À ne surtout pas modifier

Cette liste est un garde-fou contractuel : elle doit figurer dans le CDC transmis aux
prestataires.

| Élément | Interdiction | Pourquoi |
| --- | --- | --- |
| **`provenance` (Deal)** | **Ne jamais y ajouter d'options Calling / Mailing** | Décrit l'origine de l'affaire (AO, Demande entrante, Développement Commercial, Fidélisation, Écosystème), renseignée sur **465/465**, et alimente au moins 3 rapports existants. Y mêler le canal casserait ces rapports et fusionnerait deux taxonomies distinctes. |
| `raison_ce_non_presentee` | Ne pas convertir son type | HubSpot restreint les changements de type sur une propriété existante, et une conversion invalide les valeurs non conformes. Traitement prévu en §3.3. |
| Pipelines et étapes existants | Ne pas renommer, ne pas supprimer, ne pas réordonner | 465 transactions et l'ensemble des rapports en dépendent |
| Les 89 rapports enregistrés | **Cloner, jamais éditer en place** | Plusieurs servent au pilotage courant hors périmètre de ce projet |
| `motif_de_perte`, `motif_d_abandon`, `motif_de_rupture` | Hors périmètre | Aucune interaction avec les KPI de prospection |

---

## 3. Reprise des 100 deals historiques

Source : `deals-developpement-commercial-a-qualifier.csv` (100 lignes, colonnes de saisie vides).

### 3.1. Ce que les données permettent réellement

| Constat | Conséquence |
| --- | --- |
| 206 contacts associés, **tous** sans `provenance` | Aucune reprise automatique possible : il n'existe aucune valeur à recopier |
| **30 deals / 100** portent au moins une activité tracée | Seuls ceux-là sont qualifiables sur preuve |
| **70 deals / 100** sans aucune activité | Non qualifiables par les données |
| Une partie des propriétaires sont inactifs (`ref__owner_actif`) | Le vivier de personnes pouvant témoigner est réduit |

### 3.2. Stratégie en trois niveaux

**Niveau 1 — Preuve CRM (≈30 deals).** Le canal est déduit de l'activité réellement enregistrée
sur la transaction (appel logué → Calling, e-mail → Mailing).
→ `canal_prospection` renseigné, `canal_fiabilite = Preuve CRM`, `justification` = nature de la preuve.

**Niveau 2 — Déclaratif.** Le propriétaire est encore actif et se souvient du canal.
→ `canal_prospection` renseigné, `canal_fiabilite = Déclaratif`, `justification` = nom de la
personne interrogée et date.

**Niveau 3 — Non qualifié (le reste).**
→ `canal_prospection` **laissé vide**, `canal_fiabilite = Non qualifié`.

**Règle d'or** — un deal non qualifiable reste vide. Attribuer un canal au jugé produirait un
historique inventé, et fausserait durablement les taux de transformation par canal, qui sont
précisément l'objet du projet. Mieux vaut un historique partiel qu'un historique faux.

### 3.3. Traitement de `raison_ce_non_presentee`

1. Créer `motif_ce_non_presentee` (P-03).
2. Reporter à la main les **4** valeurs existantes vers la nouvelle propriété.
3. Archiver `raison_ce_non_presentee` — **archiver, pas supprimer** : réversible.

### 3.4. Mode d'import

Import CSV via l'interface (*Données → Importer → Mettre à jour des transactions existantes*),
en utilisant `hs__deal_id` comme clé de rapprochement. À ce volume, l'API n'apporte rien et
l'import UI laisse une trace consultable.

**Ne pas importer les lignes de niveau 3** : une cellule vide dans un import CSV n'efface pas
la valeur, mais la ligne n'a de toute façon rien à écrire.

---

## 4. Workflows

Deux workflows distincts plutôt qu'un seul : la dérivation de `famille_canal` doit fonctionner
aussi quand le canal est saisi **à la main**, pas seulement quand il est recopié.

### W-01 · Recopie du canal Contact → Deal

| | |
| --- | --- |
| Nom | `[KPI] Canal de prospection — recopie contact → deal` |
| Type | Basé sur les **transactions** |

**Déclencheur** — Transaction créée **OU** `provenance` du contact associé modifiée.
**Ré-inscription : à activer.** Sans cela, une transaction créée avant la qualification du
contact conservera un canal vide pour toujours.

**Action** — Copier `provenance` du **contact principal associé** vers `canal_prospection`.

**Condition** — N'écraser une valeur existante que si `canal_prospection` est vide, afin de
préserver la reprise historique et les saisies manuelles.

**Règle de désambiguïsation** — une transaction peut porter plusieurs contacts (jusqu'à 2 dans
l'export). Le contact retenu est le **contact principal** de la transaction. Cette règle doit
être écrite au CDC, sans quoi chaque prestataire l'interprétera différemment.

**Dépendance bloquante** — l'action « copier depuis un objet associé » exige une édition
**Professional ou Enterprise**. En édition Starter, W-01 est impossible : le canal devient
100 % manuel et S-02 (§5) reste le seul garde-fou.

### W-02 · Dérivation de la famille de canal

| | |
| --- | --- |
| Nom | `[KPI] Famille de canal — dérivation` |
| Type | Basé sur les **transactions** |

**Déclencheur** — `canal_prospection` est connu. Ré-inscription à activer.

**Logique** — branchement sur la valeur :

| `canal_prospection` | → `famille_canal` |
| --- | --- |
| `Cold Calling` | **Calling** |
| `Campagne mail`, `Mailing personnalisé` | **Mailing** |
| toute autre valeur | **Autre** |

Ce workflow ne dépend pas de W-01 : il fonctionne aussi bien pour une valeur recopiée que pour
une valeur saisie manuellement ou importée. **Il reste donc utile même en édition Starter.**

---

## 5. Règles de saisie obligatoire

Sans ces trois règles, tout ce qui précède produira des tableaux de bord vides. C'est le point
le plus déterminant du projet, et le moins technique.

| Réf | Règle | Où |
| --- | --- | --- |
| **S-01** | `provenance` **obligatoire à la création d'un contact** | Réglages → Objets → Contacts → Personnaliser le panneau de création |
| **S-02** | `canal_prospection` **requis pour passer à « CE en cours »** | Réglages → Objets → Transactions → Pipelines → Pré-étude → étape « CE en cours » → Propriétés requises |
| **S-03** | `motif_ce_non_presentee` **requis pour passer à « CE non-présentée »** | Idem, étape « CE non-présentée » |

S-02 place le point de contrôle au bon endroit : assez tôt pour que le canal soit encore connu
du commercial, assez tard pour ne pas alourdir la création d'une transaction.

---

## 6. Rapports et tableaux de bord

**Aucune API publique HubSpot ne permet de créer un rapport ou un tableau de bord.**
L'intégralité de cette section est manuelle, quel que soit le prestataire.

### 6.1. Rapports à cloner

Méthode identique pour chacun : ouvrir → **Cloner** → ajouter le filtre `famille_canal` →
enregistrer sur le tableau de bord R-06.

| Réf | Indicateur | Rapport de départ |
| --- | --- | --- |
| R-01 | CE signés par canal | `CE signées 25-26` |
| R-02 | CA signé par canal | `Courbe de CA` / `CA du mois` |
| R-03 | Taux de transformation par canal | `Taux de transformation - CE présentée` |
| R-04 | Panier moyen par canal | `Panier moyen - Annuel - 25-26` |

### 6.2. Rapport à créer

**R-05 — No show par canal.** Transactions dont le `dealstage` est « CE non-présentée »
**ou** dont `motif_ce_non_presentee = No show client`, groupées par `famille_canal`.

### 6.3. Tableau de bord

**R-06 — « Prospection — KPI »**, regroupant R-01 à R-05, avec filtres période, propriétaire,
secteur, `type_d_entreprise` et `type_d_etude`.

### 6.4. Deux règles de construction

- **Nommage** : préfixer tous les nouveaux rapports par `[KPI Prospection]`. Le portail en
  compte déjà 89 ; sans convention, les clones deviendront indistinguables des originaux.
- **Exclusion des non qualifiés** : chaque rapport doit soit exclure
  `canal_fiabilite = Non qualifié`, soit l'afficher en série distincte. Sans cela, les
  transactions non qualifiées seront lues comme des zéros et écraseront les taux.

### 6.5. Hors périmètre à ce stade

Les indicateurs **Nombre de personnes contactées**, **Nombre de réponses** (Calling) et
**Taux d'ouverture** (Mailing) reposent sur les objets Appel et E-mail, aujourd'hui
inaccessibles. 3 des 14 KPI ne peuvent donc pas être spécifiés — et ne doivent pas être
chiffrés au forfait.

---

## 7. Mode d'exécution par étape

| Réf | Étape | API | MCP | MANUEL |
| --- | --- | :---: | :---: | :---: |
| P-01 | Créer `canal_prospection` | **Oui** | Vérif. seule | Possible |
| P-02 | Créer `famille_canal` | **Oui** | Vérif. seule | Possible |
| P-03 | Créer `motif_ce_non_presentee` | **Oui** | Vérif. seule | Possible |
| P-04 | Créer `canal_fiabilite` | **Oui** | Vérif. seule | Possible |
| §3.3 | Migrer les 4 valeurs puis archiver | **Oui** | Vérif. seule | Recommandé (4 valeurs) |
| §3.4 | Importer la reprise des 100 deals | Oui | Vérif. seule | **Recommandé** (traçabilité) |
| W-01 | Workflow de recopie | Théorique | Non | **Recommandé** |
| W-02 | Workflow de dérivation | Théorique | Non | **Recommandé** |
| S-01 | `provenance` obligatoire à la création | Non | Non | **Obligatoire** |
| S-02 | Propriété requise à « CE en cours » | Non | Non | **Obligatoire** |
| S-03 | Propriété requise à « CE non-présentée » | Non | Non | **Obligatoire** |
| R-01→R-05 | Rapports | Non | Vérif. seule | **Obligatoire** |
| R-06 | Tableau de bord | Non | Non | **Obligatoire** |

**API — endpoints et scopes.** Propriétés : `POST /crm/v3/properties/deals`, archivage
`DELETE /crm/v3/properties/deals/{name}` — scopes `crm.schemas.deals.write` et `.read`.
Reprise de données : `POST /crm/v3/objects/deals/batch/update` — scope
`crm.objects.deals.write`. Les scopes de **schéma** sont distincts des scopes
d'**enregistrement** : écrire un deal ne donne pas le droit de créer une propriété.

**Workflows « théorique ».** `POST /automation/v4/flows` (scope `automation`, édition Pro+)
existe, mais le format JSON des flows v4 est verbeux et peu documenté : le coût de construction
par API dépasse celui de l'interface pour un résultat identique. La v3 est exclue — elle ne
gère que les workflows basés sur les contacts.

**MCP.** Les outils de cette conversation sont en lecture seule : ils ne construisent rien,
mais ils recettent tout (contrôle de type et d'options d'une propriété, taux de remplissage
avant/après, intégrité des rapports existants).

---

## 8. Checklist d'exécution

### Étape 0 — Décisions *(aucune dépendance — bloque l'étape 4)*

- [ ] Confirmer l'édition HubSpot souscrite : Starter / Pro / Enterprise *(Réglages → Compte et facturation)*
- [ ] Si Starter : acter l'abandon de W-01 et le passage au canal 100 % manuel
- [ ] Décider si `canal_fiabilite` (P-04) est retenue
- [ ] Trancher la règle « contact principal » pour W-01 et l'inscrire au CDC
- [ ] Compléter le CDC : contact commercial, date limite, planning

### Étape 1 — Saisie *(aucune dépendance — peut démarrer immédiatement)*

- [ ] **S-01** · rendre `provenance` obligatoire à la création d'un contact — MANUEL
- [ ] Communiquer la règle à l'équipe commerciale *(sans quoi elle sera contournée)*
- [ ] **MCP** · relever le taux de remplissage de `provenance` — point de référence avant/après

### Étape 2 — Propriétés *(dépend de l'étape 0 pour P-04)*

- [ ] **P-01** · créer `canal_prospection`, 7 options, valeurs identiques à `provenance` Contact
- [ ] **P-02** · créer `famille_canal`, 3 options
- [ ] **P-03** · créer `motif_ce_non_presentee`, 7 options
- [ ] **P-04** · créer `canal_fiabilite`, 3 options *(si retenue)*
- [ ] **MCP** · vérifier type et options des 4 propriétés créées
- [ ] Reporter les 4 valeurs de `raison_ce_non_presentee` → `motif_ce_non_presentee`
- [ ] Archiver `raison_ce_non_presentee` *(archiver, ne pas supprimer)*

### Étape 3 — Reprise historique *(dépend de l'étape 2)*

- [ ] Répartir les 100 lignes du CSV entre niveaux 1 / 2 / 3
- [ ] Niveau 1 · qualifier les ≈30 deals sur preuve d'activité
- [ ] Niveau 2 · interroger les propriétaires encore actifs, tracer la source en `justification`
- [ ] Niveau 3 · laisser `canal_prospection` **vide**, marquer `Non qualifié`
- [ ] Importer par `hs__deal_id` *(Données → Importer → mettre à jour l'existant)*
- [ ] **MCP** · contrôler la répartition obtenue par `famille_canal`

### Étape 4 — Automatisation *(dépend des étapes 0 et 2)*

- [ ] **W-02** · workflow de dérivation `famille_canal` — *à faire en premier, utile même en Starter*
- [ ] **W-01** · workflow de recopie contact → deal — *seulement si édition Pro+*
- [ ] Activer la **ré-inscription** sur les deux workflows
- [ ] Vérifier la condition « ne pas écraser une valeur existante » sur W-01
- [ ] **S-02** · `canal_prospection` requis à « CE en cours » — MANUEL
- [ ] **S-03** · `motif_ce_non_presentee` requis à « CE non-présentée » — MANUEL
- [ ] Tester sur une transaction de test avant activation générale

### Étape 5 — Reporting *(dépend des étapes 3 et 4)*

- [ ] **R-06** · créer le tableau de bord « Prospection — KPI »
- [ ] **R-01 → R-04** · cloner les 4 rapports + filtre `famille_canal`
- [ ] **R-05** · créer le rapport No show par canal
- [ ] Appliquer le préfixe `[KPI Prospection]` à tous les nouveaux rapports
- [ ] Exclure ou isoler `canal_fiabilite = Non qualifié` dans chaque rapport
- [ ] **MCP** · vérifier qu'aucun des 89 rapports d'origine n'a été modifié

### Étape 6 — Lot conditionnel *(dépend de l'élargissement des droits)*

- [ ] Élargir les droits du connecteur en lecture Appels / E-mails / Réunions
- [ ] Spécifier les 3 KPI d'activité restants
- [ ] Les maintenir hors du forfait tant qu'ils ne sont pas spécifiables

### Étape 7 — Clôture

- [ ] Rédiger le glossaire des 14 KPI *(définition, source, maille, propriétaire)*
- [ ] Recette — **sur le taux de remplissage, pas sur l'existence des objets**
- [ ] Former l'équipe commerciale
- [ ] Documenter la configuration livrée

---

## 9. Le point de vigilance unique

Si un seul élément de ce document devait être retenu : **`provenance` est renseignée sur
1 contact sur 13 751.** Les quatre propriétés, les deux workflows et les six rapports peuvent
être livrés à la perfection et ne produire que des tableaux de bord vides.

L'étape 1 ne dépend de rien et conditionne la valeur de toutes les autres. C'est par elle
qu'il faut commencer, avant même de créer la première propriété.
