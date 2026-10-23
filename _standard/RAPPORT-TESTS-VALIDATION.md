# Bouton « Valider » : causes réelles, correctifs et campagne de tests

**Date** : 2026-10-23 · **Branche** : `arena/ba3089c7-comparateurpro` · **Moteur** : `2026-10-23-clic1`

Ce rapport ne décrit **aucun test simulé**. Tous les résultats ci-dessous
viennent d'un **vrai navigateur** (Chromium 153 headless, piloté par CDP) qui
charge les LP servies par `_local/server.py`, tape au clavier
(`Input.insertText`, événements clavier réels), déplace une vraie souris
(`Input.dispatchMouseEvent`, hit-testing réel avec le layout réel) et exécute
les scripts de page. Les envois sont vérifiés dans `_local/leads/*.jsonl`
(mock CRM + Sheet) et dans `_local/logs/events.jsonl` (traces du moteur).

> Limite d'environnement : la sandbox n'a pas d'accès TLS sortant vers
> `mc.hellolead.pro` / `script.google.com`. Les envois sont donc validés de
> bout en bout **contre les endpoints locaux** ; le contrat de payload
> (formId, data, double transmission) est identique. Le seul test vers la
> production réellement exécuté est celui du navigateur de l'utilisateur
> (`/automobile/`, lead reçu côté CRM + Sheet).

---

## 1. Ce qui bloquait réellement

Le clic sur « Valider » ne produisait **ni envoi, ni message**. Six causes
distinctes, toutes reproduites en navigateur, se cumulaient selon les pages.

| # | Cause observée | Preuve |
|---|---|---|
| 1 | **Bouton bridé en CSS** : Formly/Webflow pose `pointer-events:none` + `opacity:.4` + classe `disabled` (sans attribut `disabled`) et le réapplique en boucle. Le clic n'atteint jamais le bouton → aucun `submit`, aucun message. | `<input class="… w-button disabled">` avec `pointer-events:none`, `opacity:0.4`, `disabled:false` ; `elementFromPoint` renvoyait `DIV.div-block-85` |
| 2 | **Clic annulé par la page** : un handler appelle `preventDefault()` sur l'événement `click`, ce qui annule l'action par défaut (la soumission du formulaire). | trace `clic-sans-soumission` avec `actionAnnuleeParLaPage: true` |
| 3 | **`form.submit()` natif** : plusieurs scripts de page soumettent par `HTMLFormElement.submit()`, qui **n'émet aucun événement `submit`** et navigue en GET. Le lead partait dans une navigation, jamais dans le CRM. | URL finale `…/page/?source=…` (les champs de tracking injectés par le moteur), aucune trace d'envoi |
| 4 | **Écouteur de capture de la page** : un script enregistre un écouteur `click` en capture sur `document` **avant** le nôtre et appelle `stopImmediatePropagation()` : notre relais `document` ne recevait plus rien. | 7 pages muettes alors que le bouton était cliquable et cliqué |
| 5 | **Message de succès invisible** : `.w-form-done` est masqué par la feuille Webflow (`display:none`) ; retirer l'attribut `hidden` ne suffit pas. Le moteur cachait le formulaire → l'utilisateur voyait un **bloc vide**. | après envoi : `form.style.display = none` et `.w-form-done` toujours `offsetHeight === 0` |
| 6 | **Marquage effacé** : `markInvalid()` posait `aria-invalid`, puis `showError()` le retirait aussitôt → aucun champ surligné. | `invalids: []` alors que le message affichait « Merci de compléter… » |

Cause annexe (environnement de test, corrigée aussi) : `_local/runtime.js`
sérialisait le `dataLayer` avec `JSON.stringify` ; les LP y poussent des
`HTMLInputElement` (inputmask) → structure circulaire → **TypeError levée
dans le script de la page**, qui interrompait son propre parcours.

---

## 2. Correctifs appliqués

Tout est dans `_standard/ubepro-lead.js`, `_standard/ubepro-ui.css` et
`_local/runtime.js`. **Aucun `style.css` de LP n'est modifié**
(`git diff --stat -- '*/style.css'` → vide) et `verify-standard.py` reste
conforme sur les 110 pages.

| Correctif | Fichier | Neutralise |
|---|---|---|
| Écoute `submit` / `click` **sur `window` en capture** (en plus de `document`) | `ubepro-lead.js` | cause 4 |
| Réactivation du contrôle : `pointer-events`, `opacity`, classe `disabled`, attribut `disabled`, + classe `.ubp-clickable` en `!important` + `MutationObserver` + rappel 700 ms | `ubepro-lead.js` + `ubepro-ui.css` (`.ubp-clickable`) | cause 1 |
| Filet de clic ramené de 350 ms à **80 ms** | `ubepro-lead.js` | cause 2 |
| Filet **`pointerdown` / `mousedown`** (250 ms) | `ubepro-lead.js` | clic consommé avant tout `click` |
| Détournement de `HTMLFormElement.prototype.submit()` vers le moteur (uniquement pour les formulaires de lead) | `ubepro-lead.js` | cause 3 |
| `.ubp-form-error` créé au besoin, `display:block !important`, `visibility:visible`, champ fautif surligné + nommé | `ubepro-lead.js` + `ubepro-ui.css` | silence de validation |
| Message explicite listant les champs manquants (« Merci de compléter les champs « Nom », « Prénom », « Téléphone »… ») | `ubepro-lead.js` | cause 6 |
| `markInvalid()` ne se fait plus effacer par `showError()` | `ubepro-lead.js` | cause 6 |
| `showSuccess()` force l'affichage de `.w-form-done` / `.form-success` | `ubepro-lead.js` | cause 5 |
| Garde d'étape sur « Suivant » : champ bloquant nommé, sinon message de repli — **jamais de bouton muet** | `ubepro-lead.js` | étapes qui ne répondent pas |
| Sérialisation circulaire défensive du `dataLayer` local | `_local/runtime.js` | TypeError dans les scripts de page |

Garde-fous conservés : périmètre `[data-api-endpoint]` et formulaires de
recherche exclus, jamais de modification de `display`/`visibility` des LP,
CSS du moteur isolé en `.ubp-`.

---

## 3. Campagne réelle : 110 LP

Méthode par page : chargement réel → fermeture des bandeaux cookies (`.ubp-` et
`#accept-all-banner-btn` d'origine) → remplissage type utilisateur → **clic
souris réel** sur « Valider » → lecture de `ubeproLead.lastOrigin`,
`lastResults`, du `dataLayer` et des blocs de message visibles.

| Résultat | Pages |
|---|---|
| **Envoi réel confirmé (Sheet `ok`) + `formulaire_lead` + message de succès visible** | **73** |
| Validation bloquante **expliquée** à l'utilisateur (message + champ surligné) | 12 |
| Formulaire **sans aucun champ de coordonnée** (quiz / routage) → hors périmètre lead | 8 |
| Étape non franchie par le pilote automatique (bouton final jamais atteint) | 16 |
| Aucun formulaire | 1 |

Détail des 73 : 63 constatés directement, + 10 pages qui **redirigent après
envoi** (`data-redirect`) et dont l'envoi est attesté par la trace
`envoi-termine` (`sheet: ok`) dans `_local/logs/events.jsonl` — la navigation
détruit le contexte JS, c'est pourquoi `lastOrigin` y apparaît vide.

Les 8 pages hors périmètre : `accueil`, `canal`, `energie-v2`,
`form-vehicules-utilitaires`, `assurance-assurance-professionnelle-iard-v2`,
`assurances-multirisques-v2`, `acces-internet-et-telephonie-pro-v2`,
`vehicules-professionnel-v2` — vérifié dans le HTML : **0 champ
`email`/`tel`/`mail`/`telephone`**.

Sur les 16 pages où le pilote n'atteint pas le bouton final, deux contrôles
manuels confirment le comportement attendu du garde d'étape :

```
securite-professionnels → clic « Suivant » sans remplir
  → « Merci de compléter le champ « Alarme anti-intrusion et télésurveillance » avant de valider »
  → champ `reponse_1_1` surligné (aria-invalid)
```

Les autres sont des étapes à choix multiples dont le pilote ne compose pas la
bonne combinaison (limite du pilote, pas du moteur).

---

## 4. Traces de référence (extraits réels)

```
# /automobile/ — cause 2 prouvée
ubepro-lead  clic-sans-soumission   { form: wf-form-Automobile, bouton: VALIDER, actionAnnuleeParLaPage: true }
ubepro-lead  submit-intercepte      { form: wf-form-Automobile, origine: filet-clic }
ubepro-lead  envoi-termine          { form: wf-form-Automobile, sheet: ok, crm: unverified }

# /edenred-ticket-restaurant-juillet-v1/ — cause 3 prouvée
ubepro-lead  soumission-programmee  { form: wf-form-edenred_formulaire_ticket }
ubepro-lead  submit-intercepte      { origine: evenement-submit }
ubepro-lead  envoi-termine          { sheet: ok, crm: unverified }   → puis data-redirect
```

`crm: unverified` = endpoint CRM de production injoignable depuis la sandbox
(pas de TLS sortant) ; l'appel part bien en `no-cors`, ce qui interdit d'en
lire la réponse — comportement attendu, identique en production.

---

## 5. Ce qui reste ouvert

1. **Pollution CRM réelle impossible depuis la sandbox** (réseau) : le
   parcours complet vers `mc.hellolead.pro` et Apps Script doit être rejoué
   depuis un navigateur hors sandbox (`?ubepro_prod=1`).
2. **16 pages non parcourues jusqu'au bouton final** par le pilote (quiz à
   combinaisons multiples) : le garde d'étape y affiche un message, mais le
   scénario « tout rempli → envoi » n'y a pas été exécuté.
3. Les **8 pages sans coordonnée** devront, si elles doivent générer des
   leads, recevoir un champ de contact ou un renvoi vers `formulaire-globale`.
