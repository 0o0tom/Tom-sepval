# E0 — Cadrage de l'outil SCI (aucun code)

Version du document : E0 r1 — 06/10/2026. Base de référence : prompt v3.
Statut : **en attente de ton « ok » ou de tes corrections**. Si tu dis « ok », je passe à E0b (spike) sans autre validation.

---

## 1. Les 6 questions (avec ma réponse par défaut)

| # | Question | Ma réponse par défaut si tu ne dis rien |
|---|---|---|
| 1 | **Biens et lots.** Combien de biens, combien de lots par bien ? Habitation, professionnel ? En copropriété ? Zone tendue ou encadrement des loyers ? | 2 biens, 1 lot d'habitation chacun, pas de local pro. Copropriété : inconnue, champ à remplir. Zone tendue : non, case à cocher par bien. |
| 2 | **Société et associés.** SCI familiale (parents et alliés jusqu'au 4e degré) ? Nombre d'associés ? Démembrement (usufruit / nue-propriété) ? Un associé personne morale ? | SCI familiale = oui. 2 à 4 associés, tous personnes physiques. Pas de démembrement. Pas de personne morale. |
| 3 | **Emprunt(s).** Combien ? Taux fixe ou variable ? Assurance prélevée avec l'échéance ou à part ? Sous quelle forme as-tu le tableau d'amortissement (PDF, Excel, papier) ? | 1 prêt amortissable, taux fixe, mensuel. Assurance prélevée **à part** (comme dans le scénario de test). Tableau en PDF : on le saisit ou on le colle, le CSV est un bonus. |
| 4 | **Revenus et exercice.** La SCI a-t-elle d'autres revenus que des loyers nus (intérêts de placement, meublé, sous-location) ? Exercice social au 31/12 ? | Uniquement des revenus fonciers → 2072-S-SD. Exercice social = année civile, clôture au 31/12. |
| 5 | **Appareils et hébergement.** Mac ou PC ? Chrome, Edge, Safari ? iPad en usage réel ou occasionnel ? Où vit le fichier : double-clic (`file://`) ou URL fixe Netlify ? | Mac + Chrome en principal, iPad Safari en lecture et encaissements ponctuels. Hébergement : **un site Netlify dédié, séparé de tomsepval.fr** (voir §2.2). Choix final après le spike E0b. |
| 6 | **Reprise de l'existant.** À partir de quelle date l'outil fait foi ? Qu'est-ce qui est déjà en cours (baux, dépôts détenus, prêt, comptes courants, quittances déjà numérotées) ? | L'outil fait foi au **01/01/2026**. J'importe les loyers et dépenses 2026 déjà passés (CSV de la banque ou saisie). L'exercice 2025 reste hors outil (déjà déclaré en mai 2026). Compteur de quittances repris au dernier numéro que tu me donnes. |

---

## 2. Mon mode de travail

### 2.1 Mode A (fichiers + exécution)

Je travaille dans un conteneur avec :
- **Node 22** : je lance les tests des fonctions pures sous Node (`vm`, sans DOM).
- **Chromium sans interface** (Playwright) : je lance l'autotest complet sur le HTML ouvert en `file://`.
- **git** : un commit et un tag par incrément, sur la branche de travail.

Ce que je **ne peux pas** exécuter, donc toujours marqué « NON VÉRIFIÉ » avec une checklist manuelle pour toi :
- Safari macOS et iPadOS (purge à 7 jours, repli téléchargement, impression).
- La vraie boîte d'impression « Enregistrer en PDF ».
- Les fenêtres de permission de File System Access (fichier de sauvegarde lié).
- **Ton** navigateur, **ton** URL Netlify.

Limite de sortie : sans objet en mode A, j'écris sur disque, jamais le fichier complet dans le chat. Si le conteneur est perdu, tu me redonnes le dernier HTML et je le re-découpe par les marqueurs `BEGIN/END`.

Organisation proposée (dans ce dépôt, branche de travail uniquement) :

```
sci-gestion/
  docs/            cadrage, bilans, PACK DE REPRISE
  src/             une source par section (SECOURS, CSS, CORE, STORAGE…)
  build/           assemble.js (+ contrôle anti-URL externe), split.js
  tests/           runner Node, runner Chromium headless
  dist/            sci-gestion-E<n>.html (on n'écrase jamais une livraison)
```

### 2.2 Point d'attention : ne pas l'héberger sur tomsepval.fr

Ce dépôt, c'est ton site vitrine. Deux raisons de **ne pas** servir l'outil depuis le même domaine :

1. **Même origine = même base.** IndexedDB est rangée par origine (domaine). Ton site charge un script tiers depuis un CDN (`cdn.jsdelivr.net/npm/@emailjs/browser`). Tout script qui tourne sur `tomsepval.fr` peut lire les bases de `tomsepval.fr`. Sur un site séparé, ce risque disparaît.
2. **Un merge sur `main` publierait l'outil** si Netlify déploie depuis ce dépôt.

Défaut : un **site Netlify dédié** (nom neutre, URL fixe), alimenté par glisser-déposer du fichier `dist/`. Idéalement un dépôt GitHub séparé aussi. Tu tranches à la question 5.

---

## 3. Schéma de données ajusté

Base `sci-le-revest-v1` (démo : `sci-demo`, tests : `sci-test`). Stores : `meta`, `records`, `blobs`, `audit`, `snapshots`.
Tous les enregistrements métier vont dans `records` avec un champ `type` (index sur `type`). Chacun porte `id` (UUID), `type`, `createdAt`, `updatedAt`, `deletedAt?`.
Montants en centimes entiers (`…C`), dates ISO `AAAA-MM-JJ`, indices × 100, taux en points de base.

### 3.1 Ajustements par rapport au §8 (à valider)

| # | Changement | Pourquoi |
|---|---|---|
| A1 | **Les lignes sont stockées dans le mouvement** (`mouvement.lignes[]`, chaque ligne a son propre `id`), pas dans un type `ligne` séparé. | Σ lignes = montant devient vérifiable en une seule écriture. Pas de ligne orpheline possible. Un mouvement validé est figé d'un bloc. Les index en mémoire (par bail, période, catégorie) donnent les mêmes requêtes. |
| A2 | **Pas de `compteCourantMvt`.** Un compte courant d'associé est un `compte` de genre `cca`. Ses mouvements sont des `mouvement` ordinaires. | Une seule mécanique (lignes, validation, numérotation, extourne). Le §5.3 le dit déjà. |
| A3 | **Le pointage est un type à part** (`pointage` : relevés[] ↔ mouvements[]), pas un champ du mouvement. | Un mouvement validé est immuable, or le pointage arrive après. Le n-n (plusieurs lignes ↔ plusieurs mouvements) tient naturellement. |
| A4 | **`avenant` et `revisionLoyer` fusionnés** en `avenant` avec `origine` = `avenant` \| `revision`. | Le dû d'une période se calcule en lisant les avenants datés. Une seule liste à parcourir. La révision garde ses champs propres (indices ancien et nouveau, trimestre). |
| A5 | **Le tableau d'amortissement est une version complète** (`tableauEmprunt` : version, date d'effet, source, `echeances[]`). Une échéance se désigne par `tableauId#n`. | 240 lignes par version, ça tient dans un enregistrement. Le remboursement anticipé crée une nouvelle version, l'ancienne reste. |
| A6 | **Le DPE est rattaché au lot**, pas au bien. | Un DPE concerne un logement. Un bien à 2 appartements a 2 DPE. Si 1 bien = 1 lot, c'est transparent. |
| A7 | `immeuble` renommé **`bien`**. | C'est le mot de l'interface. |
| A8 | Ajouts : `regleCategorisation`, `mappageBanque`, `lotImport`, `reportAlerte`, `relance`, `reprise`. | Besoins des §5.3, §5.8, §7 non couverts par le §8. |

### 3.2 Liste des types

**Société**
- `societe` : denomination, siren?, siret?, siege, dateCreation, capitalC, nbPartsTotal, familiale (bool), regime `IR`, clotureSociale (`MM-JJ`), gerants[].
- `associe` : nom, prenom, genre `PP` \| `PM`, adresse, regimeInfo `reel` \| `micro` \| `inconnu`, beneficiaireImposable (texte libre).
- `mouvementParts` : date, nature (`souscription` \| `cession` \| `donation` \| `autre`), cedantId?, cessionnaireId?, nbParts, prixC?.

**Biens et baux**
- `bien` : nom, adresse, surface, annee, acquisition {date, prixC, notaireC, droitsC}, valeurEstimeeC?, zoneTendue, zoneEncadrement, copropriete, archive.
- `lot` : bienId, nom, typeLocal `habitation` \| `professionnel`, surface, cleRepartition?.
- `diagnostic` : lotId, genre (`DPE`, …), classe?, dateRealisation, dateValidite.
- `locataire` : nom, prenom, email?, tel?, garant? (texte), accordDemat (bool), anonymiseLe?.
- `bail` : lotId, type `habitation` \| `professionnel`, locataireIds[], dateDebut, dureeMois, loyerHCC, provisionC, provisionTeomC?, complementC?, forfaitCharges (bool), depotContractuelC, jourEcheance, terme `echoir` \| `echu`, modePaiement, clauseIndexation, familleIndice, indiceRef {trimestre, valeur}, dateRevision (`MM-JJ`), statut, sortie? {dateCongeRecu, preavisMois, dateRemiseCles, edlConforme, retardImputableLocataire}.
- `avenant` : bailId, dateEffet, origine `avenant` \| `revision`, nouvelles valeurs (loyerHCC, provisionC…), revision? {indiceAncien, indiceNouveau, trimestre}.
- `indice` : famille (`IRL`, `ILC`, `ILAT`), trimestre (`2025-T2`), valeur ×100, datePublication?, saisiLe.

**Comptabilité**
- `compte` : nom, genre `banque` \| `cca`, associeId? (si cca), ibanMasque?, soldeReprise {date, montantC}.
- `categorie` : code, libelle, nature (`RECETTE`, `CHARGE_DEDUCTIBLE`, `CHARGE_RECUPERABLE`, `NON_DEDUCTIBLE`, `HORS_RESULTAT`), modeRecup `BRUT` \| `NET`, exigeBien, regleTravaux?, ligne2072 {annee: libellé}, statut.
- `mouvement` : compteId, date, montantC (signé), libelle, statut `brouillon` \| `valide`, numero? ({annee, n}), source (`saisie`, `releve`, `loyers`, `emprunt`, `od`, `reprise`), extourneDe?, extourneePar?, payeur?, pieceIds[], commentaire, **lignes[]** : {id, montantC, categorieId, bienId?, lotId?, bailId?, associeId?, imputation? {bailId, periode `AAAA-MM`, composante}, echeanceRef?}.
- `ligneReleve` : compteId, date, montantC, libelle, empreinte, rangOccurrence, lotImportId. Immuable.
- `pointage` : releveIds[], mouvementIds[], date.
- `regleCategorisation`, `mappageBanque`, `lotImport` (reprise et relevés, annulable).
- `emprunt` : libelle, compteId, capitalC, tauxBp, dureeMois, dateDebut, differe?, assuranceMensuelleC?.
- `tableauEmprunt` : empruntId, version, dateEffet, source `calcul` \| `saisie` \| `csv`, echeances[] {n, date, capitalC, interetsC, assuranceC, fraisC, crdC}.
- `exercice` : annee, verrouJusquau?, versionsCloture[] {v, date, empreinteBalance, empreinteJournal, motifReouverture?}.
- `reprise` : dateFoi, statut `brouillon` \| `validee`, soldes (comptes, dépôts, CRD, CCA), reportANouveauC.

**Loyers et documents**
- `quittance` : numero (`Q-AAAA-NNNN`), genre `quittance` \| `recu`, bailId, periode, contenuFige (montants, texte, locataires), versionModele, emiseLe, annuleeLe?.
- `relance` : bailId, periodes[], niveau 1/2/3, date.
- `regularisationCharges` : bailId, annee, detail par nature, soldeC, communiqueLe.
- `piece` : nom, mime, taille, sha256, rattachements[]. Le binaire est dans `blobs` sous le même id.
- `document` : titre, tags[], pieceId?, rattachements[].
- `contestationTaxe` : impot, bienId, annee, dateMiseEnRecouvrement, dateReclamation, statut.
- `assemblee` : date, genre, ordreDuJour[], decisions[], statut.
- `parametre` : cle, annee?, valeur, unite, misAJourLe, source, statut (`indicatif` \| `confirme`), confirmeLe?.
- `reportAlerte` : cleAlerte, jusquau.

**meta** : schemaVersion, instanceId, revision, compteurs {quittance: {AAAA: n}, mouvement: {AAAA: n}}, bailVerrou {ongletId, expireA}, dernieresSauvegardes, fichierLie (handle).

### 3.3 Contraintes vérifiées dans `repo` (pas seulement à l'écran)
Bail → lot existant. Pas deux baux actifs qui se chevauchent sur un lot. Σ lignes = montant. Mouvement validé immuable (sauf pièces, commentaire). Pas d'écriture ≤ date de verrou ni avant la date de reprise. Pas de suppression physique d'une entité référencée. Dépôt habitation ≤ 1 mois HC (paramètre). Avertissement si Σ parts ≠ capital.

---

## 4. Liste des écrans

Navigation latérale. Entre parenthèses : l'incrément qui livre l'écran.

1. **À faire** — accueil (E12, conçu dès E4)
2. **Mois en cours** — encaisser en 1-2 clics, panier d'impression (E6, E7)
3. **Tableau de bord** (E12)
4. **Société** — fiche SCI, associés, parts, gérants (E4)
5. **Biens et lots** — avec diagnostics (E4)
6. **Locataires et baux** — fiches, avenants, révision IRL guidée (E4, E8), fin de bail (E8)
7. **Indices** — table IRL/ILC/ILAT saisie par toi (E4)
8. **Loyers** — échéancier, compte locataire, impayés (E6), quittances et relances (E7)
9. **Comptabilité** — journal, saisie ventilée, catégories, comptes, comptes courants (E5)
10. **Pièces jointes** (E5)
11. **Emprunts** (E9)
12. **Banque** — import de relevés, pointage, rapprochement (E10)
13. **Contrôles et clôture** (E10, E11)
14. **Fiscalité** — résultat, quotes-parts, déficits, 2072-S-SD, régularisation de charges, taxes et contestations, calendrier (E11)
15. **Rapports** — journal, grand livre, balance, état des loyers…, dossier comptable (E5 → E11)
16. **Vie sociale** — AG, registres (E13)
17. **Documents** — bibliothèque, modèles de courriers, export `.ics` (E13)
18. **Paramètres** — SCI, `PARAMS`, sauvegarde/import, fichier lié, démo, reprise de l'existant, réinitialisation, audit, sécurité, « Hypothèses et limites », « Reprise et mise à jour » (E2, E3, E5, E11)
19. **Diagnostics** — autotests, intégrité, erreurs, rapport copiable, export brut de secours (E1)

---

## 5. Budget de lignes

**Problème à trancher.** Les budgets du §10.7 font 9 150 lignes au total (E1 à E14). Ça dépasse le budget global de ~8 000 lignes et le seuil d'alerte de 9 000 du §10.6 avant même la première ligne écrite.

Ma proposition : compter les **autotests à part** (section `AUTOTESTS`, ils ne servent qu'au diagnostic) et serrer le code applicatif grâce au moteur générique.

| Incrément | §10.7 | Proposé (code appli) | Tests (à part) |
|---|---:|---:|---:|
| E0b spike (jetable) | 150 | 150 | — |
| E1 socle | 700 | 650 | 200 |
| E2 stockage, audit, params, démo | 700 | 650 | 120 |
| E3 sauvegarde, import, migrations | 800 | 700 | 120 |
| E4 fiches | 900 | 650 | 100 |
| E5 mouvements, pièces, reprise | 900 | 850 | 100 |
| E6 loyers, échéancier | 600 | 500 | 80 |
| E7 quittances, impression | 500 | 450 | 60 |
| E8 IRL, fin de bail, dépôt | 600 | 450 | 80 |
| E9 emprunts | 400 | 350 | 60 |
| E10 banque, clôture | 700 | 650 | 80 |
| E11 charges, fiscalité, 2072 | 900 | 850 | 200 |
| E12 tableau de bord, À faire | 450 | 400 | 40 |
| E13 vie sociale, documents | 600 | 450 | 40 |
| E14 chiffrement, finitions | 400 | 350 | 60 |
| **Total** | **9 150** | **≈ 7 950** | **≈ 1 340** |

Fichier final estimé : ~9 300 lignes avec les tests. Le seuil de 9 000 s'appliquerait au code appli (7 950). Si tu préfères compter les tests dedans, il faudra activer le §10.6 tôt (réduire E13/E14).

---

## 6. Tests par incrément

IDs stables préfixés par famille (le prompt utilise `D1` à la fois pour les dates et les déficits : je préfixe tout pour éviter la collision).

| Incr. | Tests |
|---|---|
| E1 | `MONEY-M1…M5` · `PARSE-01…11` (7 acceptés dont U+00A0 et U+202F, 4 refusés) · `DATE-01…09` (fins de mois, 29/02, terme de bail, préavis, horloge injectable avec un générateur d'alerte fictif) · `VALID-01…07` (SIREN, SIRET, IBAN, longueur) · `SPLIT-S1…S5` · `HASH-01…04` (vide, abc, Web Crypto = JS pur, clés désordonnées = même empreinte canonique) · `CSV-01…05` · `INTEG-01…03` (section manquante, version incohérente, fin absente → lecture seule) · `OFFLINE-01` (contrôle d'assemblage : aucune URL externe, aucun `fetch`/XHR) |
| E2 | `STORE-01…06` · `PARAMS-01…03` (valeurs initiales « indicatif », date et statut affichés, aucun calcul avec une valeur vide) · `DEMO-01` · `AUDIT-01…03` (annulation, compteur jamais reculé, événements critiques non purgés) |
| E3 | `HASH-05` (exportedAt et revision exclus) · `IMPORT-01…05` · `MIGR-E2` (fixture E2) |
| E4 | `FICHE-01…08` (SIREN invalide, Σ parts ≠ capital, lot avec bail non supprimable, 6 ans si non familiale, bail pro sans règles habitation, chevauchement, dépôt > 1 mois refusé, recherche) · `MIGR-E3` |
| E5 | `FLUX-01…04` · `CSV-06…07` (journal, Excel FR) · `PIECE-01…03` (NDJSON aller-retour, SHA-256 vérifié, pièce manquante) · `REPRISE-01…02` (à blanc, annulation du lot) · `MIGR-E4` |
| E6 | `LOYER-L1…L8` (L6 : imputation et reste ; la décision reçu/quittance est en E7) · `ECH-01` (avenant sans perte de paiement) · `MIGR-E5` |
| E7 | `QUIT-01…04` · `LOYER-L6b` (reçu puis quittance) · `QUIT-05` (colocation) · impression 4 pages : **manuel** · `MIGR-E6` |
| E8 | `IRL-01…08` · `DEPOT-01…07` · `DATE-10` (alerte IRL réelle à la bonne date) · `MIGR-E7` |
| E9 | `LOAN-01…08` · `MIGR-E8` |
| E10 | `BANK-01…05` · `CLOT-01…03` · `MIGR-E9` |
| E11 | `FISC-F1…F7` (dont F1b) · `DEF-D1…D3` · `E2E-SCEN` (§11.2 au centime) · `E2E-01, 02, 03, 04` · `MIGR-E10` |
| E12 | `DASH-01…04` (attendus, encaissés, avance, rendement 4,31 %) · `TRESO-01` (5 537,95 €) · `MIGR-E11` |
| E13 | `BACKUP-01…02` (JSON + NDJSON sur base vide = même empreinte ; pièce retirée listée) · `DOC-01` (recherche par tag) · `MIGR-E12` |
| E14 | `CRYPT-01…03` · `PERF-01` (< 3 s, mesuré) · `MIGR-E13` · rejeu de tous les tests précédents |

À chaque incrément : `DEMO-n` (le générateur de démo produit le contenu attendu de l'incrément).

---

## 7. Ce que j'ai déjà vérifié (exécuté sous Node, pas supposé)

J'ai recalculé les vecteurs chiffrés du §11 avec un script jetable (centimes entiers, arrondi demi à l'écart de zéro). **Tous cohérents** :
- SIREN, SIRET, IBAN : acceptés / refusés comme attendu.
- Prêt 100 000 € à 2 % sur 240 mois : 505,88 € ; éch. 1 = 166,67 + 339,21 ; éch. 240 = 0,84 + 506,07 ; intérêts totaux 21 412,23 € ; 12 premières échéances : 1 962,49 € d'intérêts, 4 108,07 € de capital.
- Prorata L1, L3, L4, répartition S4, IRL 665,60 €.
- Scénario §11.2 : recettes 13 994,51 €, charges 5 748,49 €, résultat 8 246,02 €, A 4 947,61 / B 3 298,41, banque 6 737,95 €, trésorerie disponible 5 537,95 €.

Ce n'est pas un autotest de l'outil (il n'existe pas encore). C'est juste la preuve que les vecteurs sont justes avant de coder dessus.

---

## 8. Hypothèses, ambiguïtés et petits conflits relevés

1. **Assurance emprunteur à part** des échéances de 505,88 € : c'est ce que donne le calcul des sorties du §11.2 (3 786 + 6 070,56 + 5 000).
2. **Test horloge `[E1]`** : l'alerte « révision IRL » n'existe qu'en E8. En E1 je teste le mécanisme (horloge injectable + registre d'alertes) avec une alerte fictive. Le vrai cas IRL arrive en E8 (`DATE-10`).
3. **Dépôt > 1 mois HC** : demandé en E4 (« Fini si ») et en E8 (T-DEPOT). Le blocage est livré en E4, le test E8 le rejoue.
4. **`T-PARAMS` « aucune valeur légale en dur »** : un test ne peut pas deviner qu'un nombre est « légal ». Méthode : toutes les valeurs légales vivent dans une table unique `VALEURS_INITIALES` ; le test vide `PARAMS` et vérifie que chaque calcul concerné se **bloque** avec un message au lieu de tourner avec une valeur cachée.
5. **Valeurs initiales** : je mettrai dans `VALEURS_INITIALES` seulement les valeurs déjà écrites dans ton prompt (3 ans, 6 ans, 1 mois, 10 %, 20 %, 3 mois / 1 mois, 6 mois, 6 ans de conservation…), toutes en statut « indicatif, non confirmé ». Le plafond d'imputation du déficit foncier, les délais de réclamation par impôt, le calendrier DPE et la date limite 2072 : **je les laisse vides** sauf si tu me les donnes. Tu les remplis et tu coches « confirmé ».
6. **Numérotation des mouvements** par année civile ; celle des quittances aussi (`Q-AAAA-NNNN`), avec l'année de **la période quittancée**. À confirmer : ou l'année d'émission ?
7. **Safari** : pas de File System Access → sauvegarde par téléchargement uniquement, rappels renforcés. C'est le repli prévu.

---

## 9. Prochaine étape

Tu réponds aux 6 questions (ou juste « ok »). Je lance **E0b** : un fichier jetable de ~150 lignes que tu ouvres dans **ton** navigateur, en double-clic puis hébergé. Il affiche un tableau OK/KO et un bouton « Copier le rapport ». Tu me colles le rapport, je t'en déduis le mode d'usage recommandé.

Je n'écris aucune ligne de l'application avant ton feu vert.
