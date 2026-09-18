# Plan d'implémentation — Suivi des KPI de prospection

Portail `27023376` — IÉSEG CONSEIL Paris (EU1, EUR, Europe/Paris).
Établi le 18 septembre 2026, en lecture seule. **Aucune modification n'a été faite dans HubSpot.**

Documents sources : note de cadrage du 17 septembre 2026 et `audit-kpi-prospection.md`.

---

## 0. Contexte technique vérifié

### 0.1. Le dépôt Git n'est pas le projet

Le dépôt contient un site statique de 145 lignes (`index.html` + 6 feuilles CSS + 4 images),
sans build ni backend, et sans aucune intégration HubSpot. **Aucune modification de code n'est
requise par ce projet** : tout se joue dans le portail.

Une seule remarque connexe : la capture de leads du site passe par un embed **Typeform**
(`data-tf-widget="DwEUoIT8"`), qui n'écrit pas dans HubSpot. Hors périmètre du CDC, mais à
signaler — un lead entrant par ce canal n'existe pas dans le CRM et n'a donc pas de `provenance`.

### 0.2. L'édition souscrite reste inconnue

`accountType` renvoie `STANDARD`, ce qui désigne la **catégorie** de compte (par opposition à
sandbox/développeur), **pas le niveau d'abonnement**. L'exigence T-02 — Starter / Pro /
Enterprise — n'est donc toujours pas tranchée, et aucun appel en lecture ne permet de la
trancher. Elle doit être confirmée par un administrateur dans
*Réglages → Compte et facturation*.

C'est la dépendance n°1 du projet : elle conditionne le workflow de recopie (§3) et le
Custom Report Builder multi-objets.

### 0.3. Droits du connecteur — inchangés après reconnexion

| Objet | Lecture | Écriture |
| --- | --- | --- |
| Contact, Company, Deal, Ticket | OK | Réautorisation requise |
| Call, Email, Meeting, Note, Task | Réautorisation requise | Réautorisation requise |
| Campaign | Montée d'édition requise | Montée d'édition requise |

Aucun objet n'est accessible en écriture. Et surtout, le connecteur MCP **n'expose aucun outil
de création de propriétés, de workflows ou de dashboards** : c'est une limite fonctionnelle de
l'intégration, que l'élargissement des droits ne lèvera pas.

---

## 1. Liste exhaustive des modifications HubSpot

Chaque ligne porte un identifiant repris dans le plan d'exécution (§5).

| Réf | Modification | Objet | Réf. CDC |
| --- | --- | --- | --- |
| **M-01** | Créer la propriété `canal_prospection` (liste, 7 options) | Deal | F-02, F-05 |
| **M-02** | Créer la propriété `famille_canal` (liste : Calling / Mailing / Autre) | Deal | F-04 |
| **M-03** | Remplacer `raison_ce_non_presentee` (texte) par une propriété liste | Deal | F-06 |
| **M-04** | Migrer les 4 valeurs existantes de `raison_ce_non_presentee` | Deal | F-09 |
| **M-05** | Archiver l'ancienne propriété texte après migration | Deal | F-09 |
| **M-06** | Rendre `provenance` obligatoire à la création de contact | Contact | F-09 |
| **M-07** | Rendre `canal_prospection` requise à l'étape « CE en cours » | Pipeline | F-03 |
| **M-08** | Rendre le motif requis à l'étape « CE non-présentée » | Pipeline | F-06 |
| **M-09** | Qualifier rétroactivement les 100 deals « Développement Commercial » | Deal | F-09 |
| **M-10** | Workflow de recopie `provenance` (Contact) → `canal_prospection` (Deal) | Automatisation | F-03, F-07 |
| **M-11** | Branchement workflow alimentant `famille_canal` | Automatisation | F-04 |
| **M-12** | Cloner 4 rapports existants + filtre canal (CE signés, CA, taux de transfo, panier moyen) | Reporting | F-04, F-10 |
| **M-13** | Créer le rapport No show par canal | Reporting | F-06 |
| **M-14** | Créer le tableau de bord « Prospection — KPI » | Reporting | F-10 |
| **M-15** | Élargir les droits du connecteur (lecture Appels/E-mails/Réunions) | Admin | T-02 |
| **M-16** | Spécifier les 3 KPI d'activité (contactés, réponses, taux d'ouverture) | Conception | F-02 |
| **M-17** | Glossaire des 14 KPI (définition, source, maille, propriétaire) | Documentation | F-08 |
| **M-18** | Dédoublonnage / nettoyage de la base contacts | Qualité | F-09 |
| **M-19** | Recette, formation de l'équipe, documentation technique | Conduite | T-04, T-06 |

---

## 2. Répartition par moyen d'exécution

### 2.1. Faisable avec les outils MCP actuels — lecture seule

Aucune des 19 modifications. Les outils MCP disponibles ne permettent **que de lire**.

Ils restent utiles pour piloter le projet, et c'est leur seul rôle ici :

| Usage | Outil MCP |
| --- | --- |
| Contrôler qu'une propriété a bien été créée et typée | `get_properties`, `search_properties` |
| Mesurer le taux de remplissage avant/après (recette de M-06, M-09) | `query_crm_data` |
| Extraire la liste des 100 deals à qualifier pour M-09 | `search_crm_objects`, `query_crm_data` |
| Vérifier qu'aucun rapport existant n'a été cassé | `manage_saved_reports` (LIST/FETCH) |
| Contrôler les droits après réautorisation (M-15) | `get_user_details` |

**Conclusion** : le MCP sert à *auditer et recetter*, jamais à *construire*.

### 2.2. Réalisable via l'API HubSpot

| Réf | Modification | Faisable par API |
| --- | --- | --- |
| M-01, M-02, M-03 | Création de propriétés | Oui — Properties API |
| M-04, M-09 | Migration et reprise de données | Oui — Objects ou Imports API |
| M-05 | Archivage de propriété | Oui — Properties API |
| M-10, M-11 | Workflows | Oui, sous conditions — Automation API v4 |
| M-15 | Lecture des engagements | Oui, une fois les scopes accordés |

### 2.3. Obligatoirement dans l'interface HubSpot

| Réf | Modification | Pourquoi l'UI est obligatoire |
| --- | --- | --- |
| M-06 | Propriété obligatoire à la création de contact | Configuration du panneau de création, non exposée par API |
| M-07, M-08 | Propriétés requises par étape de pipeline | La configuration « propriétés requises » d'une étape n'est pas exposée de façon fiable par l'API Pipelines |
| M-12, M-13, M-14 | Rapports et tableaux de bord | **HubSpot ne fournit aucune API publique de création de rapports ou de dashboards.** Point dur du projet. |
| M-15 | Réautorisation du connecteur | Action d'administrateur |
| M-17, M-18, M-19 | Glossaire, nettoyage, formation | Travail humain |

La ligne M-12/M-13/M-14 mérite d'être relevée dans le CDC : **tout le lot reporting est
manuel**, quelle que soit la sophistication du prestataire. Aucune automatisation n'est
possible sur cette partie.

---

## 3. Détail des API requises

### 3.1. Authentification

**Application privée** (recommandé) : *Réglages → Intégrations → Applications privées →
Créer*. Fournit un token `Bearer` unique, avec des scopes cochés explicitement.
À privilégier sur OAuth pour un projet interne sans redistribution.

Hôte API : `api.hubapi.com`. Le portail est hébergé en **UE** (`app-eu1.hubspot.com`) — à
signaler au prestataire pour la conformité RGPD et la localisation des traitements.

Limites d'appel : de l'ordre de 100 requêtes / 10 s (190 en Pro+). Dimensionner les reprises
de données (M-09) par lots plutôt qu'en appels unitaires.

### 3.2. Propriétés — M-01, M-02, M-03, M-05

| | |
| --- | --- |
| Créer | `POST /crm/v3/properties/deals` |
| Modifier | `PATCH /crm/v3/properties/deals/{propertyName}` |
| Lire | `GET /crm/v3/properties/deals` |
| Archiver | `DELETE /crm/v3/properties/deals/{propertyName}` |
| **Scopes** | `crm.schemas.deals.write` et `crm.schemas.deals.read` |

À noter : les scopes de **schéma** (`crm.schemas.*`) sont distincts des scopes
d'**enregistrement** (`crm.objects.*`). Accorder l'écriture sur les deals ne donne pas le droit
de créer une propriété — c'est une confusion fréquente dans les devis.

### 3.3. Reprise de données — M-04, M-09

| | |
| --- | --- |
| Mise à jour par lots | `POST /crm/v3/objects/deals/batch/update` (100 enregistrements max) |
| Alternative import | `POST /crm/v3/imports` |
| **Scopes** | `crm.objects.deals.write` ; `crm.import` pour la voie import |

Pour M-09 (100 deals), l'import CSV via l'interface reste la voie la plus simple et la plus
traçable ; l'API n'apporte rien à ce volume.

### 3.4. Workflows — M-10, M-11

| | |
| --- | --- |
| Créer | `POST /automation/v4/flows` |
| **Scope** | `automation` |
| **Édition** | Professional ou Enterprise obligatoire |

Trois réserves sérieuses :

1. L'ancien endpoint `/automation/v3/workflows` ne gère **que les workflows basés sur les
   contacts**. Le workflow requis ici est **basé sur les transactions** : il impose la v4.
2. Le format JSON des *flows* v4 est verbeux et peu documenté. Construire ce workflow par API
   coûte plus cher que de le faire à la souris, pour un résultat identique.
3. En édition Starter, l'action « copier une propriété depuis un objet associé » n'existe pas —
   ni par API, ni par l'interface.

**Recommandation** : faire M-10 et M-11 dans l'interface, malgré la disponibilité théorique de
l'API.

### 3.5. Lecture des engagements — M-15, M-16

| | |
| --- | --- |
| Appels | `GET /crm/v3/objects/calls` |
| E-mails | `GET /crm/v3/objects/emails` |
| Réunions | `GET /crm/v3/objects/meetings` |
| **Scopes probables** | `crm.objects.calls.read`, `crm.objects.emails.read`, `crm.objects.meetings.read` ; `sales-email-read` pour le contenu et les ouvertures d'e-mails |

Les noms exacts de scopes sont à confirmer dans l'écran de configuration de l'application
privée : HubSpot a fait évoluer cette nomenclature, et les recopier depuis une documentation
ancienne est une source d'erreur classique.

Pour le **taux d'ouverture** de campagnes marketing : `GET /marketing/v3/emails/statistics`,
scope `marketing-email`, qui suppose un Marketing Hub actif.

### 3.6. Ce qu'aucune API ne permet

- Création ou modification de **rapports personnalisés**.
- Création ou modification de **tableaux de bord**.
- Configuration des **propriétés requises par étape de pipeline**.
- Configuration du **panneau de création** d'un objet.

---

## 4. Révisions apportées à l'audit du 18 septembre

Deux points de `audit-kpi-prospection.md` sont corrigés ici.

**§3.3 — conversion de `raison_ce_non_presentee`.** L'audit recommandait de convertir la
propriété texte en liste déroulante. C'est déconseillé : HubSpot restreint les changements de
type sur une propriété existante, et une conversion réussie invalide les valeurs qui ne
correspondent à aucune option. Le volume concerné étant négligeable (4 enregistrements),
la voie sûre est : **créer une nouvelle propriété liste** `motif_ce_non_presentee`, recopier
les 4 valeurs, puis archiver l'ancienne. C'est réversible, et cela ne touche à aucun historique.
Les étapes M-03 / M-04 / M-05 reflètent cette révision.

**§1 — édition souscrite.** L'audit laissait entendre qu'un appel en lecture pourrait la
déterminer. Ce n'est pas le cas : `accountType` ne renvoie que la catégorie de compte.

---

## 5. Plan d'exécution

### Étape 0 — Décisions bloquantes *(aucune dépendance)*

- Confirmer l'édition souscrite (T-02) → conditionne E3.
- Décider si le connecteur est réautorisé, et avec quels scopes (M-15).
- Trancher le CDC : contact commercial, date limite, planning (échéance dépassée).

*Sortie : go/no-go sur le workflow automatique.*

### Étape 1 — Qualité de saisie *(aucune dépendance — peut démarrer immédiatement)*

- M-06 · `provenance` obligatoire à la création de contact.
- M-03, M-04, M-05 · nouvelle propriété motif, migration, archivage.
- M-18 · dédoublonnage.

*Cette étape ne dépend ni de l'édition, ni des droits du connecteur. C'est la seule qui
produise de la valeur sans prérequis — et celle qui conditionne la valeur de toutes les autres.*

### Étape 2 — Propriétés canal *(dépend de E0 uniquement pour l'arbitrage API/UI)*

- M-01 · `canal_prospection`.
- M-02 · `famille_canal`.
- M-17 · glossaire des 14 KPI (à mener en parallèle).

### Étape 3 — Automatisation *(dépend de E0 et E2)*

- M-10, M-11 · workflow de recopie et branchement.
- M-07, M-08 · propriétés requises par étape.

*Si l'édition est Starter : M-10 et M-11 sont abandonnés, le canal est saisi manuellement, et
M-07 devient le seul garde-fou.*

### Étape 4 — Reprise d'historique *(dépend de E2)*

- M-09 · qualification des 100 deals « Développement Commercial ».

*À faire après E2 (la propriété doit exister) mais avant E5 (sinon les dashboards démarrent
sans historique).*

### Étape 5 — Reporting *(dépend de E3 et E4)*

- M-12 · clonage des 4 rapports + filtre canal.
- M-13 · rapport No show.
- M-14 · tableau de bord.

### Étape 6 — Lot conditionnel *(dépend de M-15)*

- M-16 · spécification des 3 KPI d'activité.

*À isoler contractuellement : tant que les objets Appel et E-mail sont inaccessibles, ces
3 indicateurs sur 14 ne peuvent pas être spécifiés, donc pas chiffrés au forfait.*

### Étape 7 — Clôture

- M-19 · recette, formation, documentation.

---

## 6. Risques

| # | Risque | Gravité | Parade |
| --- | --- | --- | --- |
| R1 | **Le remplissage, pas l'outillage.** `provenance` est renseignée sur 1 contact / 13 751. Tout l'édifice peut être construit et ne produire que des dashboards vides. | **Critique** | E1 avant tout le reste ; recette chiffrée sur le taux de remplissage, pas sur l'existence des objets |
| R2 | Édition Starter → le workflow M-10 est impossible, la saisie du canal devient 100 % manuelle | Élevée | Trancher T-02 en E0 ; prévoir le scénario dégradé au contrat |
| R3 | Un prestataire « simplifie » en ajoutant Calling/Mailing aux options de `provenance` (Deal) — ce qui casse 3 rapports existants et mélange deux taxonomies | Élevée | Inscrire l'interdiction explicitement au CDC |
| R4 | Le workflow recopie `provenance` à la création du deal ; si le contact est qualifié après, le deal garde une valeur vide | Moyenne | Activer la ré-inscription (re-enrollment) sur le déclencheur |
| R5 | Deal associé à plusieurs contacts → la règle « contact principal » est ambiguë | Moyenne | Fixer la règle au CDC (contact principal, ou premier associé) |
| R6 | Reprise M-09 : réattribuer un canal à 100 deals passés relève souvent de la reconstitution | Moyenne | N'attribuer que sur preuve (activité tracée) ; laisser vide sinon, plutôt que de fabriquer un historique |
| R7 | Conversion de type sur `raison_ce_non_presentee` → perte des valeurs non conformes | Moyenne | Écartée par la révision §4 : nouvelle propriété plutôt que conversion |
| R8 | 89 rapports existants → prolifération et confusion après clonage | Faible | Convention de nommage préfixée, ex. `[KPI Prospection] …` |
| R9 | La réautorisation en écriture du connecteur est large et peu granulaire | Faible | N'accorder que les scopes de lecture nécessaires à M-16 |
| R10 | 3 KPI sur 14 non spécifiables → risque de forfait sur un périmètre indéfini | Élevée | Lot conditionnel séparé (E6) |

---

## 7. Récapitulatif

- **0 modification sur 19** réalisable avec les outils MCP actuels.
- **9 modifications** réalisables par API (dont 2 déconseillées par API — les workflows).
- **10 modifications** obligatoirement dans l'interface, dont **l'intégralité du lot reporting**.
- **1 décision** bloque le chantier d'automatisation : l'édition souscrite.
- **1 risque domine** tous les autres : la saisie, pas la technique.
