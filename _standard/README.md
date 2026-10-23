# `_standard/` — standard de production Ubepro / ComparateurPro

Portage du standard de la landing page `recrutement_it_tech` (dépôt
`servicegraphique/LP-`) sur **les 110 landing pages du dépôt qui portent un
formulaire**, sans modifier leur design.

Le CSS de chaque site (`style.css`) n'est **jamais** touché. Tout le visuel
ajouté (bannière cookies, modale) vit dans `ubepro-ui.css`, préfixé `.ubp-`.

---

## 1. Contenu

| Fichier | Rôle |
|---|---|
| `ubepro-lead.js` | Moteur de lead : lecture du formulaire, validation téléphone/CP, double transmission **CRM + Google Sheet**, événement `dataLayer` `formulaire_lead`. |
| `ubepro-consent.js` | CMP RGPD : bannière + modale de préférences, cookie `user_cookie_preferences` (180 j, `SameSite=Lax`), mise à jour du **Consent Mode v2**, purge des données de tracking non consenties. |
| `ubepro-ui.css` | Styles de la bannière et de la modale, **uniquement `.ubp-*`**. |
| `sites.json` | Configuration générée par site : `formId`, domaine, canonical, endpoints. |
| `tools/apply-standard.py` | Outil d'injection idempotent (marqueurs `<!-- ubepro:head -->` / `<!-- ubepro:body -->`) + génération `robots.txt`, `sitemap.xml`, `llms.txt`. |

## 2. Ce qui est injecté dans chaque page

Dans le `<head>`, juste après `<meta charset>` :

1. détection du mode local (`window.__UBEPRO_LOCAL__`) ;
2. **Consent Mode v2** en `denied` par défaut (`security_storage: granted`,
   `wait_for_update: 500`) ;
3. **GTM `GTM-53SH7D7F`**, chargé **uniquement hors localhost** ;
4. `Content-Security-Policy` adaptée (GTM, CRM, Apps Script, fonts locales) ;
5. **JSON-LD `@graph`** : `Organization`, `WebSite`, `WebPage`,
   `BreadcrumbList`, `Service`, et `FAQPage` si la page contient des
   `<details><summary>` ;
6. `ubepro-ui.css`.

Avant `</body>` :

1. `window.UBEPRO_STANDARD` (config du site : `formId`, endpoints, URLs légales) ;
2. `<noscript>` GTM ;
3. `ubepro-consent.js` puis `ubepro-lead.js`, en `defer`.

Et à la racine du site : `robots.txt` (avec ouverture explicite à GPTBot,
OAI-SearchBot, ClaudeBot, PerplexityBot, Google-Extended, Applebot-Extended,
CCBot), `sitemap.xml`, `llms.txt` (fiche structurée générée depuis le contenu
de la page).

## 3. Comment est calculé le `formId`

`ADW_LP_<SLUG>` par défaut (`assurance-tns` → `ADW_LP_ASSURANCE_TNS`).
La valeur réellement utilisée est celle de `_standard/sites.json`, que l'on peut
surcharger site par site avant de relancer l'outil.

## 4. Utilisation

```bash
# aperçu (aucune écriture)
python3 _standard/tools/apply-standard.py --dry-run

# application sur les 110 pages
python3 _standard/tools/apply-standard.py --apply

# une seule page
python3 _standard/tools/apply-standard.py --apply --only assurance-tns
```

L'outil est **idempotent** : il retire d'abord toute injection précédente
(marqueurs, ancien GTM, ancienne CSP) avant de réécrire. Deux passages
produisent exactement le même fichier.

**Ordre recommandé du pipeline** sur une nouvelle LP :

```bash
python3 _local/tools/localize.py --apply      # neutralise les dépendances externes
python3 _standard/tools/apply-standard.py --apply
```

`localize.py` détecte le marqueur `<!-- ubepro:head -->` et **ne touche plus**
aux blocs GTM posés par le standard.

## 5. Comportement du moteur de lead

- intercepte la soumission **au niveau `document`** (capture), donc fonctionne
  aussi avec les formulaires multi-étapes Webflow ;
- ne prend la main que si le formulaire est **valide** et **contient des
  données exploitables** ; sinon la page garde la main (aucune régression) ;
- **ignore** les formulaires portant `data-api-endpoint` (intégration dédiée,
  ex. `assurance-estheticiennes-institut-beaute` qui gère son propre parcours
  OTP + double envoi) ;
- résout les variantes de noms de champs Webflow (`telephone-2`, `Email-2`,
  `Code-postal`, `Nom-2`…) vers les clés canoniques ;
- envoie **en parallèle** vers le CRM (`sendBeacon` puis `fetch no-cors`) et
  vers Google Apps Script (urlencoded, 15 s) ; le Sheet est bloquant, le CRM
  reste « non vérifiable » (CORS) par construction ;
- affiche le succès existant de la page (`.w-form-done`, `.form-success`),
  `data-redirect` si présent, sinon un bandeau `.ubp-inline-success`.

## 6. Consentement

Les préférences sont lues dans le cookie `user_cookie_preferences` et
retransmises à `gtag('consent','update', …)` à chaque changement. Tant que
l'internaut n'a pas accepté, `analytics_storage`, `ad_storage`, `ad_user_data`
et `ad_personalization` restent à `denied`, et les données de tracking stockées
en session (`ubepro_tracking_v1`) sont purgées.

Événements émis :

| Événement | Portée |
|---|---|
| `ubepro:cookie-preferences` | `window` — préférences enregistrées |
| `ubepro:consent-changed` | `document` — même charge utile |
| `cookie_preferences_updated` | `dataLayer` |
| `formulaire_lead` | `dataLayer` — après chaque envoi réussi |

N'importe quel élément portant `id="open-cookie-preferences-btn"`,
`data-cookie-manage` ou la classe `.btn-cookie-link` rouvre la modale.

## 7. Tests

- **Audit dépendances** : `python3 _local/tools/audit.py` → 0 URL externe
  fonctionnelle hors GTM (autorisé).
- **Test fonctionnel** : `bash tester-local.sh`, puis ouvrir
  `http://localhost:8080/<slug>/` ; les deux envois sont visibles dans
  `_local/leads/leads.jsonl` (CRM) et `_local/leads/sheets.jsonl` (Sheet).
- **Campagne automatisée (jsdom)** : 109/110 pages valident le parcours complet
  (bannière, cookie, soumission, `sheet: ok`, événement `formulaire_lead`).
  La 110<sup>e</sup>, `assurance-estheticiennes-institut-beaute`, utilise son
  intégration dédiée et n'est volontairement pas interceptée.
