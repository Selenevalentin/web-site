# Suivi des KPI de prospection — audit vérifié & spécification d'implémentation

Portail HubSpot `27023376` — audit réalisé le 18 septembre 2026 en lecture seule.
Ce document complète et **corrige sur quatre points** la note de cadrage du 17 septembre 2026.

---

## 1. Ce que l'audit a réellement trouvé

L'audit initial a été refait propriété par propriété. Quatre constats de la note sont à réviser
avant toute mise en œuvre, parce qu'ils changent la conception.

### 1.1. La propriété « canal » existe déjà — côté Contact

La note indique qu'« aucune propriété ne distingue le canal, ni sur les Contacts ni sur les Deals ».
C'est inexact pour les Contacts : la propriété `provenance` existe et ses options couvrent
déjà le besoin de l'exigence F-05.

| Option `provenance` (Contact) | Famille de canal |
| --- | --- |
| Cold Calling | Calling |
| Campagne mail | Mailing |
| Mailing personnalisé | Mailing |
| Linkedin | Autre |
| Événement | Autre |
| Écosystème | Autre |
| Réseau personnel | Autre |

**Conséquence** : il ne faut pas créer une propriété « canal de prospection » sur le Contact.
Elle existe. Le chantier n'est pas de la créer, mais de la **faire remplir** (cf. 1.3).

### 1.2. `provenance` existe aussi sur le Deal — mais ne veut pas dire la même chose

Une propriété homonyme existe sur l'objet Deal, avec des options entièrement différentes :

| Option `provenance` (Deal) | Nb de deals |
| --- | --- |
| AO (`AO CNJE`) | 119 |
| Demande entrante | 111 |
| Développement Commercial | 100 |
| Fidélisation (`Ancien Client`) | 99 |
| Écosystème | 36 |
| **Total** | **465 (100 % renseignés)** |

Cette propriété décrit l'**origine de l'affaire**, pas le canal de prospection. Elle est
renseignée sur la totalité du portefeuille et alimente déjà plusieurs rapports enregistrés
(« Provenance - CA », « Provenance - CE signée - TA », « Provenance - Panier moyen »).

**Conséquence** : ne pas la détourner ni y ajouter des options Calling/Mailing — cela casserait
trois rapports existants et mélangerait deux taxonomies. Le canal doit être une **nouvelle**
propriété Deal, distincte.

Le sous-ensemble à qualifier est identifié : les **100 deals « Développement Commercial »**
sont ceux issus de la prospection sortante, donc ceux à répartir entre Calling et Mailing.

### 1.3. Le vrai point de blocage est le remplissage, pas la modélisation

| Propriété | Renseignée | Sur | Verdict |
| --- | --- | --- | --- |
| `provenance` (Contact) | **1** | 13 751 | Inexploitable en l'état |
| `provenance` (Deal) | 465 | 465 | Fiable |
| `date_de_presentation_ce` | 244 | 465 | Exploitable |
| `raison_ce_non_presentee` | **4** | 465 | Inexploitable en l'état |

La note présente `raison_ce_non_presentee` comme « une base exploitable pour le KPI No show ».
Avec 4 enregistrements renseignés sur 465, et un type **texte libre** (pas une liste d'options),
elle ne l'est pas : elle ne peut être ni agrégée ni filtrée de façon fiable.

C'est le constat structurant de cet audit : **créer des propriétés et des workflows ne produira
aucun KPI tant que la saisie n'est pas contrainte.** Le chantier prioritaire est la discipline
de saisie, pas l'outillage.

### 1.4. Le reporting n'est pas à construire de zéro

La note conclut qu'« aucune propriété de calcul des 14 KPI n'existe » et que le suivi est
entièrement manuel. En réalité le portail compte **89 rapports enregistrés**, dont plusieurs
couvrent déjà une partie des 14 indicateurs :

- `Taux de transformation - CE présentée` (et sa version 25-26)
- `CE signées`, `CE présentées`, `CE perdues`
- `Courbe de CA`, `CA du mois`, `Panier moyen - Annuel - 25-26`
- `Provenance - CA`, `Provenance - CE signée - TA`, `Provenance - Panier moyen`

**Conséquence** : la Phase 4 est un travail de **clonage + ajout d'un filtre canal** sur des
rapports existants, pas une construction ex nihilo. Le chiffrage à demander aux prestataires
doit être revu à la baisse en conséquence.

---

## 2. Ce qui bloque l'exécution directe

Les modifications ne peuvent pas être appliquées depuis cette conversation, pour deux raisons
distinctes — la seconde n'étant pas levée par un simple élargissement de droits.

1. **Droits du connecteur.** Contacts, Companies, Deals et Tickets sont accessibles en lecture
   seule. Aucun objet n'est accessible en écriture. Les objets Appel, E-mail et Réunion ne sont
   pas même lisibles (`REQUIRES_REAUTHORIZATION`), et Campagne exige une montée d'édition
   (`REQUIRES_ACCOUNT_MODIFICATION`).
2. **Périmètre de l'intégration.** Le connecteur HubSpot ne propose aucun outil de création de
   **définitions de propriétés**, de **workflows** ou de **tableaux de bord** — seulement la
   lecture/écriture d'enregistrements. Même après réautorisation en écriture, la création des
   propriétés et des workflows décrits ci-dessous devra se faire **dans l'interface HubSpot**
   ou via l'API, pas depuis cette conversation.

Le chapitre 3 est donc écrit comme une spécification directement applicable par la personne qui
tiendra la souris — interne ou prestataire.

---

## 3. Spécification d'implémentation

### 3.1. Propriété à créer : canal de prospection (Deal)

*Réglages → Propriétés → Objet « Transaction » → Créer une propriété*

| Réglage | Valeur |
| --- | --- |
| Libellé | Canal de prospection |
| Nom interne | `canal_prospection` |
| Groupe | Informations sur la transaction |
| Type | Liste déroulante (énumération), sélection unique |
| Description | Canal par lequel le contact à l'origine de l'affaire a été touché. Distinct de « Provenance », qui décrit l'origine de l'affaire. |

Options — reprises **à l'identique** de `provenance` (Contact) pour que la recopie soit sans perte :

`Cold Calling` · `Campagne mail` · `Mailing personnalisé` · `Linkedin` · `Événement` ·
`Écosystème` · `Réseau personnel`

### 3.2. Propriété à créer : famille de canal (Deal)

Les tableaux de bord demandés par le CDC raisonnent en Calling vs Mailing, pas en sept options.
Une seconde propriété évite un regroupement manuel dans chaque rapport.

| Réglage | Valeur |
| --- | --- |
| Libellé | Famille de canal |
| Nom interne | `famille_canal` |
| Type | Liste déroulante, sélection unique |
| Options | `Calling` · `Mailing` · `Autre` |

Règle de calcul (appliquée par le workflow 3.5) :

- `Cold Calling` → **Calling**
- `Campagne mail`, `Mailing personnalisé` → **Mailing**
- toute autre valeur → **Autre**

### 3.3. Propriété à corriger : raison de CE non présentée

`raison_ce_non_presentee` est aujourd'hui en texte libre et ne peut pas alimenter le KPI No show.

*Réglages → Propriétés → `raison_ce_non_presentee` → Modifier*

- Convertir le type **texte** → **liste déroulante**
- Options proposées : `No show client` · `Reporté à l'initiative du client` ·
  `Reporté à notre initiative` · `Annulée — projet abandonné` · `Annulée — budget` ·
  `Contact injoignable` · `Autre`
- Reprendre à la main les 4 valeurs existantes avant conversion (volume négligeable)

Le KPI **No show** se lit alors : deals dont `raison_ce_non_presentee = No show client`,
ou dont le `dealstage` est « CE non-présentée » (`1862597857`) — cette étape existe déjà
dans le pipeline et n'est pas à créer.

### 3.4. Rendre la saisie obligatoire

Sans cette étape, rien de ce qui précède ne produira de KPI.

- `provenance` (Contact) : marquer comme **obligatoire dans les formulaires de création**
  de contact (*Réglages → Objets → Contacts → Personnaliser le panneau de création*).
- `canal_prospection` (Deal) : rendre obligatoire au passage à l'étape **« CE en cours »**
  (*Réglages → Objets → Transactions → Pipelines → Pré-étude → étape « CE en cours » →
  Propriétés requises*).
- `raison_ce_non_presentee` : rendre obligatoire au passage à l'étape **« CE non-présentée »**.
- Reprise de l'historique : les 100 deals « Développement Commercial » doivent être qualifiés
  rétroactivement en Calling/Mailing, sans quoi les dashboards démarreront sans historique.
  À traiter par export/import CSV plutôt qu'à la main.

### 3.5. Workflow de recopie du canal

*Automatisation → Workflows → Créer → Basé sur les transactions*

**Nom** — `[KPI] Canal de prospection — recopie contact → deal`

**Déclencheur** — Transaction créée, OU `provenance` du contact associé modifiée.

**Actions**

1. Copier la valeur de `provenance` du **contact principal associé** vers
   `canal_prospection` (Deal).
2. Branchement sur `canal_prospection` pour alimenter `famille_canal` selon la règle 3.2.

**Point d'attention** — l'action « copier une valeur de propriété d'un objet associé » exige
une édition **Pro ou Enterprise**. C'est le point à trancher au titre de l'exigence T-02
avant de lancer la Phase 3 : en édition Starter, ce workflow n'est pas réalisable et le canal
devra être saisi manuellement sur la transaction.

### 3.6. Rapports

Cinq des sept indicateurs par canal se déduisent de rapports déjà en place. Pour chacun :
ouvrir le rapport existant → **Cloner** → ajouter un filtre `famille_canal` → enregistrer en
deux exemplaires (Calling / Mailing) sur un nouveau tableau de bord « Prospection — KPI ».

| Indicateur (CDC ch. 5) | Rapport de départ | Action |
| --- | --- | --- |
| CE signés | `CE signées 25-26` | Cloner + filtre canal |
| CA signé | `Courbe de CA` / `CA du mois` | Cloner + filtre canal |
| Taux de transformation | `Taux de transformation - CE présentée` | Cloner + filtre canal |
| Panier moyen par canal | `Panier moyen - Annuel - 25-26` | Cloner + filtre canal |
| No show | — | À créer : deals en « CE non-présentée », groupés par `famille_canal` |

Les indicateurs **Nombre de personnes contactées**, **Nombre de réponses** (Calling) et
**Taux d'ouverture** (Mailing) reposent sur les objets Appel et E-mail, aujourd'hui inaccessibles.
Ils ne pourront être spécifiés précisément qu'une fois l'accès à ces objets rétabli — ils sont
donc à isoler dans un lot distinct, et non à inclure dans un forfait ferme.

---

## 4. Plan d'action révisé

Le découpage de la note reste valable ; l'ordre change. La qualité de saisie passe de la Phase 5
à la Phase 1, parce qu'elle conditionne la valeur de tout le reste.

| Phase | Contenu | Réf. CDC | Préalable |
| --- | --- | --- | --- |
| 1 — Décision | Trancher l'édition HubSpot souscrite (bloque 3.5) ; élargir les droits du connecteur | T-02 | — |
| 2 — Saisie | Rendre `provenance` obligatoire ; convertir `raison_ce_non_presentee` en liste ; qualifier les 100 deals historiques | F-09, F-05 | — |
| 3 — Propriétés | Créer `canal_prospection` et `famille_canal` (3.1, 3.2) | F-02, F-05 | Phase 1 |
| 4 — Automatisation | Workflow de recopie ; propriétés requises par étape | F-03, F-06, F-07 | Phases 1 et 3 |
| 5 — Reporting | Cloner les rapports existants + filtre canal ; dashboard dédié | F-04, F-10 | Phase 4 |
| 6 — Lot conditionnel | KPI Appels/E-mails (contactés, réponses, taux d'ouverture) | F-02 | Accès Appel/E-mail |

La Phase 2 peut démarrer immédiatement : elle ne dépend ni de l'édition souscrite, ni des droits
du connecteur.

---

## 5. Points à finaliser dans le CDC avant diffusion

- Page de garde : nom et coordonnées du contact commercial manquants.
- Chapitre 12 : adresse e-mail de réponse et date limite manquantes.
- Chapitre 9 : le planning vise une remise « avant septembre » alors que le document date du
  21 août 2026. L'échéance est dépassée — à remplacer par une date réaliste.
- Chapitres 10 et 12 (budget, modalités) volontairement ouverts : à confirmer que c'est bien
  l'intention pour un appel à propositions.
- À ajouter au vu de cet audit : préciser que 89 rapports existent déjà et que le lot reporting
  consiste à les adapter, afin que les propositions ne soient pas chiffrées comme une
  construction complète.
