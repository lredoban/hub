# Veillance — Cerveau métier, spec moteur, tests, MVP, moat, validation

*7 juillet 2026 · basé sur : RGAA 4.1.2 (106 critères, RGAA 5 attendu fin 2026), WCAG 2.1/2.2 AA, EAA + loi FR (décrets 2023), jurisprudence Carrefour (TJ Caen, juin 2026), étude marché du 07/07 (Siteimprove 12-30 k$/an, Access42 1 800-5 600 €/audit, trou de marché agences FR).*

---

## 1. Carte complète du domaine

### 1.1 Concepts métier

| Concept | Définition opérationnelle | Piège |
|---|---|---|
| **RGAA** | Référentiel FR : 106 critères en 13 thématiques (images, cadres, couleurs, multimédia, tableaux, liens, scripts, éléments obligatoires, structuration, présentation, formulaires, navigation, consultation). Chaque critère = tests précis. Version 4.1.2 ; **RGAA 5 attendu fin 2026** — ton produit doit versionner son référentiel. | Le RGAA n'est pas WCAG traduit : c'est une **méthode de test** avec des règles d'applicabilité. Dire « conforme WCAG » ≠ « conforme RGAA ». |
| **Taux de conformité** | % = critères conformes / critères **applicables** (les non-applicables sortent du calcul). Se mesure sur un **échantillon** de pages défini par la méthode (accueil, contact, mentions, plan du site, aide, authentification, pages représentatives…). | Un taux « site entier » n'existe pas réglementairement. Ton produit calcule un taux *indicatif automatisé* — vocabulaire à verrouiller partout. |
| **Déclaration d'accessibilité** | Document obligatoire : état de conformité (total/partiel/non conforme), résultats d'audit, contenus non accessibles + dérogations, schéma pluriannuel (3 ans) + plan d'action annuel. | **Une déclaration exige un audit manuel complet.** Ton produit aide à la préparer/maintenir, ne la génère jamais « automatiquement conforme ». |
| **Automatisable vs manuel** | ~30 % des critères pleinement automatisables (axe-core & co), ~10 % partiels, ~60 % humains (pertinence des alternatives, ordre de lecture, compréhension). | LE plafond du produit. Chaque rapport doit afficher « X critères vérifiés automatiquement, Y nécessitent une revue humaine » — sinon sur-promesse = responsabilité. |
| **Obligés** | EAA (depuis 28/06/2025) : entreprises > 10 salariés OU > 2 M€ CA pour e-commerce, banque, télécom, transport, médias, livres numériques. Secteur public : obligation historique (taux + déclaration + schéma). Déclaratif renforcé > 250 M€ CA. | Confusion massive 2 M€ vs 250 M€ dans le marché — l'expliquer honnêtement = crédibilité (et disqualifier les prospects non obligés plutôt que leur vendre de la peur). |
| **Sanctions & risque réel** | Cadre : jusqu'à 50 k€/service (manquements EAA), contraventions cumulables ; moteur actuel = **contentieux associatif civil** (Intérêt à Agir : Auchan/Carrefour/Leclerc/Picard assignés, Carrefour condamné juin 2026). | Vendre « le précédent Carrefour », pas « l'amende qui pleut » (elle ne pleut pas encore). |
| **Régression** | Une mise en prod qui casse un critère jusque-là conforme (alt perdus, contraste du nouveau bouton, focus supprimé). | C'est LA proposition de valeur monitoring : l'audit manuel est une photo, toi tu es la vidéo. |
| **Overlay** | Widgets « accessibilité en 1 script » (accessiBe…) — détestés de la communauté a11y, juridiquement fragiles. | Ne jamais en vendre, ne jamais y ressembler. Positionnement anti-overlay explicite = crédibilité communautaire. |

### 1.2 Acteurs

- **Client payeur** : le patron/lead dev d'**agence web FR** (5-50 pers., 10-80 sites clients en maintenance) qui revend ×2-3.
- **Utilisateurs** : chef de projet agence (rapports, alertes), dev agence (findings + fix), le client final de l'agence (lecteur du rapport white-label).
- **Écosystème** : cabinets d'audit (Access42, Temesis — **partenaires potentiels**, pas concurrents : tu leur amènes le manuel), auditeurs freelances, ARCOM/DGCCRF, assos requérantes, DINUM (Ara, référentiel).

### 1.3 Données nécessaires

Par site : URL racine, sitemap ou liste de pages, périmètre d'échantillon (8-15 pages type méthode RGAA), credentials éventuels (espace connecté), fréquence de scan, seuils d'alerte. Par agence : marque blanche (logo, couleurs), liste sites/clients, destinataires d'alertes. Par scan : DOM rendu (navigateur headless), captures, résultats axe-core par règle, mapping règle→critère RGAA, diff vs scan précédent.

### 1.4 Règles produit

1. **Mapping axe-core → RGAA** : chaque règle axe est rattachée à un critère RGAA + niveau de confiance (`auto` / `partiel` / `signal`). Une règle `partiel` ne peut produire qu'un warning « à vérifier », jamais un « conforme ».
2. **Trois états par critère** : `non-conforme détecté` (preuve machine), `rien détecté` (≠ conforme !), `non testable automatiquement`. Interdit d'afficher « conforme » sur la seule absence d'erreur machine.
3. **Score produit** : « score de risque automatisé » 0-100 par page/site (pondéré par impact utilisateur : bloquant lecteur d'écran > cosmétique) — nom volontairement distinct du « taux de conformité RGAA ».
4. **Régression** = apparition d'un finding absent du scan N-1 sur la même page → alerte immédiate (email/Slack), avec le commit/date de détection.
5. **Échantillonnage** : scanner tout le sitemap mais rapporter par échantillon méthode + top offenders.

### 1.5 Exceptions & cas limites

SPA à rendu client (attendre hydration, states des routes) · contenus derrière login (2FA, RGPD des credentials) · cookie walls/CMP qui masquent la page · iframes tiers (YouTube, cartes — critères applicables mais hors contrôle du client → statut « tiers ») · PDF téléchargeables (obligation réelle, non scannable par axe → signaler « X PDF non audités ») · médias (sous-titres : détectable en partie, qualité non jugeable) · contenus dérogeables (archives, cartes interactives) · sites multilingues (lang, échantillon par langue) · webfonts/contrastes calculés sur images de fond (faux négatifs) · rate-limit/WAF qui bloquent le crawler · site du client de l'agence qui interdit le scan (autorisation contractuelle à prévoir dans les CGV agence).

### 1.6 Zones d'incertitude

Sortie et contenu exact du RGAA 5 (fin 2026) · doctrine de contrôle ARCOM/DGCCRF (volume réel de contrôles privés) · appétit des agences à payer vs sous-traiter aux cabinets (hypothèse H1 commerciale) · position des assos sur les outils automatiques (s'en faire des alliées, pas des ennemies).

### 1.7 À ne JAMAIS promettre

1. « Conformité garantie / site 100 % accessible » — impossible par construction (60 % manuel).
2. « Déclaration d'accessibilité générée automatiquement » — tu génères un **brouillon pré-rempli** clairement marqué « à compléter par audit manuel ».
3. « Protège des poursuites » — tu réduis le risque de régression, point.
4. Aucun conseil juridique (obligé/pas obligé = analyse au cas par cas → renvoyer vers avocat/cabinet).
5. Jamais le mot « certifié ».
→ CGV : obligation de moyens ; le client final reste responsable de sa déclaration ; scan uniquement sur sites autorisés.

---

## 2. Spécification fonctionnelle du moteur

### 2.1 Modules

```
crawler/    — découverte pages (sitemap + BFS limité), rendu Playwright, captures
analyzer/   — axe-core + checks maison (contrastes calculés, lang, titres, focus visible)
mapping/    — règles → critères RGAA (fichier versionné, cœur propriétaire)
differ/     — diff findings scan N vs N-1 → régressions/résolutions
reporter/   — rapport HTML/PDF white-label + brouillon de déclaration + export CSV
alerter/    — email/Slack sur régression, digest hebdo
api/        — REST + webhooks ; dashboard Nuxt minimal
```

### 2.2 Inputs / Outputs

**Input scan** : `{siteId, pages[]|sitemap, auth?, viewport[], depth}`. **Output** :

```ts
{
  site: 'clientX.fr', scanId, date, referential: 'RGAA 4.1.2 / axe-core 4.10 / mapping 2026.07',
  summary: {
    riskScore: 62,                       // 0-100, automatisé
    criteria: { failed: 14, passedAuto: 31, needsHuman: 61 },   // total 106
    regressions: 3, resolved: 5, pagesScanned: 42, pagesFailed: ['upload PDF x12 non audités']
  },
  findings: [{
    critere: '3.2',                      // contrastes
    level: 'auto'|'partiel',
    severity: 'bloquant'|'majeur'|'mineur',
    page: '/checkout', selector: '.btn-primary',
    message: "Contraste 2.8:1 entre le texte du bouton « Payer » et son fond (minimum : 4.5:1).",
    fix: "Passer le fond de #7BA7E0 à #3D6DB5 conserve votre teinte et atteint 4.6:1.",
    screenshot: '…', firstSeen: '2026-07-01', status: 'new'|'recurring'|'regression'
  }],
  humanChecklist: [{ critere: '1.1', question: "Les 17 images détectées avec alt ont-elles des alternatives PERTINENTES ?" }],
  disclaimer: "Analyse automatisée ~30-40 % des critères RGAA. Ne remplace pas un audit manuel ni ne permet une déclaration de conformité."
}
```

### 2.3 Décisions & niveaux de confiance

Verdicts par critère : `FAIL` (preuve machine, screenshot à l'appui) · `PASS_AUTO` (uniquement pour les critères 100 % automatisables) · `NEEDS_HUMAN` (défaut pour 60 % des critères — jamais silencieux, toujours listés avec la question à se poser) · `NOT_APPLICABLE` (heuristique + confirmable par l'humain) · `BLOCKED` (page inaccessible au scan : auth, WAF, timeout → refus de conclure, jamais compté conforme).
Erreurs moteur distinctes des findings a11y : `E-CRAWL-*` (robots.txt interdit, 4xx/5xx, JS crash), `E-AUTH-*`, `E-SCOPE-*` (domaine non autorisé — blocage dur).

### 2.4 API / écrans

```
POST /v1/sites             créer un site (agence → N sites)
POST /v1/sites/:id/scan    lancer un scan (+ cron interne hebdo)
GET  /v1/sites/:id/report/latest   (JSON | HTML | PDF white-label)
GET  /v1/sites/:id/diff?from&to
POST /v1/webhooks          (régressions → Slack/email)
Écrans V1 (Nuxt) : liste sites (score, tendance, régressions) · détail site (findings filtrables, checklist humaine) · réglages marque blanche · gestion clés/quota.
```

---

## 3. Corpus de tests (52 scénarios)

**Crawl & rendu (CR)**
1. CR-01 · sitemap.xml propre 40 pages · 40 pages scannées, échantillon RGAA identifié.
2. CR-02 · pas de sitemap · BFS depth 3, cap 100 pages, homepage+mentions+contact trouvées.
3. CR-03 · SPA Nuxt/React · findings identiques sur 3 runs (attente hydration stable, pas de flaky).
4. CR-04 · cookie wall CMP · bannière détectée, scan post-consentement simulé + la bannière elle-même auditée (critère focus/contraste).
5. CR-05 · page 500 · `BLOCKED`, pas de faux « 0 erreur ».
6. CR-06 · robots.txt disallow · scan refusé par défaut, override explicite avec case « j'ai l'autorisation du propriétaire ».
7. CR-07 · WAF rate-limit · backoff, scan partiel signalé `BLOCKED: 12 pages`.
8. CR-08 · redirection vers domaine hors périmètre · E-SCOPE, on ne suit pas.
9. CR-09 · espace connecté (form login fourni) · scan OK, credentials chiffrés, jamais dans les logs.
10. CR-10 · site 5 000 pages · échantillon + cap documenté, durée < 15 min, coût borné.

**Détection a11y (A)**
11. A-01 · `<img>` sans alt · FAIL critère 1.1, sélecteur + screenshot.
12. A-02 · alt="image1.jpg" (alt pourri) · `NEEDS_HUMAN` avec question ciblée — PAS un PASS (axe ne voit rien).
13. A-03 · contraste 2.8:1 sur bouton · FAIL 3.2 avec suggestion de couleur atteignant 4.5:1 conservant la teinte.
14. A-04 · texte sur image de fond CSS · `partiel` (contraste non calculable) → NEEDS_HUMAN, pas de faux FAIL.
15. A-05 · `<html>` sans lang · FAIL 8.3.
16. A-06 · hiérarchie h1→h3 sans h2 · FAIL 9.1.
17. A-07 · formulaire input sans label · FAIL 11.1.
18. A-08 · placeholder utilisé comme label · FAIL 11.1 avec fix (label visible ou aria-label).
19. A-09 · focus outline supprimé (`outline:none`) globalement · FAIL 10.7 (le check maison, axe le rate souvent).
20. A-10 · lien « cliquez ici » ×8 · `NEEDS_HUMAN` 6.1 avec liste des liens.
21. A-11 · vidéo YouTube embarquée · `NEEDS_HUMAN` 4.x + statut « contenu tiers ».
22. A-12 · carrousel auto sans pause · FAIL 13.8.
23. A-13 · aria-hidden="true" sur du contenu interactif · FAIL (piège classique).
24. A-14 · tableau de données sans th/scope · FAIL 5.x.
25. A-15 · zoom 200 % casse le layout (overflow) · FAIL 10.4 via viewport test.
26. A-16 · page parfaite (fixture) · 0 FAIL, ~31 PASS_AUTO, 61 NEEDS_HUMAN listés — le rapport ne dit JAMAIS « conforme ».
27. A-17 · PDF liés (12) · non audités, comptés et signalés dans le résumé.
28. A-18 · page en anglais sur site FR · lang incohérent détecté (8.7/8.8).

**Régressions & diff (R)**
29. R-01 · scan N : bouton contrasté ; scan N+1 : contraste cassé · finding `regression`, alerte < 5 min.
30. R-02 · finding corrigé · `resolved`, compté dans le digest positif (l'agence montre sa valeur au client).
31. R-03 · page supprimée du site · findings archivés, pas comptés comme resolved.
32. R-04 · nouvelle page ajoutée · findings `new`, distingués des régressions.
33. R-05 · même finding, sélecteur légèrement changé (refonte CSS) · dédup par empreinte (critère+page+contexte), pas de double alerte.
34. R-06 · site refondu à 100 % · seuil « diff massif » → un email de synthèse, pas 400 alertes.
35. R-07 · flapping (finding qui apparaît/disparaît selon le run) · marqué `unstable`, exclu des alertes, signalé.

**Rapports & white-label (W)**
36. W-01 · rapport PDF logo agence · aucune mention Veillance si plan white-label.
37. W-02 · brouillon de déclaration généré · contient les blocs réglementaires + bandeau rouge « à compléter par audit manuel » non supprimable.
38. W-03 · export CSV findings · importable dans un ticketing.
39. W-04 · digest hebdo multi-sites · 1 email, sites triés par régressions.
40. W-05 · rapport d'un scan `BLOCKED` partiel · le % non scanné est en première page (jamais caché).

**Multi-tenant, quotas, sécu (S)**
41. S-01 · agence 40 sites plan 25 · blocage soft + upsell, pas de scan silencieusement sauté.
42. S-02 · clé API révoquée · 401 immédiat.
43. S-03 · un membre d'agence ne voit pas les sites d'une autre (tenant isolation, test automatisé).
44. S-04 · credentials de login site chiffrés au repos (KMS) · vérif + jamais en clair dans Sentry.
45. S-05 · URL interne (http://192.168…, localhost, métadonnées cloud) · **refus SSRF** — test sécurité bloquant.
46. S-06 · scan demandé sur un domaine gouvernemental sans autorisation · refus + message (politique d'usage).

**Ambigus & refus de conclure (X)**
47. X-01 · « quel est mon taux de conformité RGAA ? » (question produit) · le produit affiche « score de risque automatisé 62/100 ; le taux RGAA exige un audit manuel — voici le brouillon + 2 cabinets partenaires ».
48. X-02 · site conforme à 100 % selon axe · verdict global reste « aucune non-conformité détectée automatiquement (≈35 % des critères) ».
49. X-03 · client demande « suis-je obligé ? (CA 1,8 M€) » · réponse produit : arbre de décision informatif + « consultez un conseil » — jamais oui/non.
50. X-04 · demande de suppression du disclaimer dans le rapport white-label · impossible (choix produit assumé, protège l'agence aussi).
51. X-05 · RGAA 5 publié · les scans historiques restent taggés 4.1.2, les nouveaux basculent après opt-in agence, diff inter-référentiels refusé.
52. X-06 · page avec 2 000 erreurs (site cassé) · cap d'affichage + « site probablement en erreur, scan à relancer », pas un rapport de 400 pages.

---

## 4. MVP vendable

### 4.1 V1 indispensable

1. Crawler Playwright + axe-core + 5 checks maison (focus visible, lang, contrastes sur rendu, PDF count, titres).
2. **Mapping axe→RGAA versionné** avec les 3 statuts (FAIL / PASS_AUTO / NEEDS_HUMAN) — le cœur.
3. Diff + alertes régression email.
4. Rapport HTML/PDF **white-label** + brouillon de déclaration (bandeau obligatoire).
5. Dashboard minimal (liste sites, détail findings) + Stripe (par paliers de sites).
6. **Scan gratuit une page** en ligne (machine à leads, même mécanique que le validateur FacturX).

### 4.2 Simulable manuellement

Le tri des findings et le premier rapport de chaque pilote (toi, 1 h/site — c'est aussi ta formation continue) · la checklist humaine remplie en option payante (149 €/site, sous-traitable plus tard à un auditeur freelance) · l'onboarding (visio 30 min) · la veille réglementaire (newsletter manuelle).

### 4.3 Inutile avant les premiers paiements

App mobile, SSO, scan continu temps réel, corrections automatiques (danger overlay !), RGAA 5 anticipé, intégrations CI/CD, multi-langues d'interface, IA générative de fixes (plus tard, avec garde-fous), API publique documentée complète.

### 4.4 Architecture minimaliste

```
Monorepo pnpm TS. Workers de scan : conteneur Playwright (Railway) + queue (pg-boss sur Postgres — pas de Redis).
API + dashboard : Nuxt (tu es chez toi) + Nitro server routes.
DB Postgres : agencies, users, sites, scans, findings (JSONB), subscriptions, alerts.
Stockage screenshots : S3-compatible (Scaleway), TTL 90 j.
Coût de scan borné : budget CPU/page + cap pages = COGS prévisible par palier.
```

### 4.5 Ordre exact de build (~3 semaines)

J1-2 crawler + rendu stable (CR-01→03) → J3-4 axe-core + mapping RGAA v1 (30 critères les plus fréquents) + fixtures A-\* → J5 checks maison → J6-7 modèle findings + diff (R-01→05) → J8 rapport HTML white-label → J9 brouillon déclaration + disclaimers → J10 dashboard minimal → J11 alertes + digest → J12 Stripe paliers → J13 scan gratuit public → J14-15 sécurité (S-05 SSRF, chiffrement) → J16-21 pilotes assistés à la main.

---

## 5. Moat solo-dev

- **Open source** : les checks maison individuels (focus, lang…) en petits packages — crédibilité communauté a11y FR (elle est petite, influente, anti-overlay : soit elle t'adopte, soit elle t'enterre).
- **Payant** : le pipeline complet (crawl+diff+rapports+alertes), le mapping axe→RGAA maintenu, le white-label.
- **Difficile à copier** : le **mapping RGAA à 3 statuts avec messages/fixes en français** (nourri par chaque site scanné), le corpus de fixtures de sites cassés, la relation avec les cabinets (apporteur de manuel ↔ apporteur d'outil).
- **Source de vérité** : publier un **Observatoire mensuel de l'accessibilité** (score automatisé des 100 plus gros e-commerçants FR — même mécanique que le 32 % de Leclerc cité partout) → PR, SEO, backlinks, et personne ne peut le publier sans l'outil.
- **Crédibilité** : disclaimer inviolable (protège l'agence ET toi) · partenariats affichés avec 1-2 auditeurs certifiés pour le manuel · transparence méthodo (page « ce qu'on ne peut PAS tester ») — la modestie technique est LE signal de sérieux dans ce milieu.

---

## 6. Plan de validation commerciale

### 6.1 Dix prospects

1-4. **Agences WordPress/e-commerce FR de 10-30 pers.** avec offre de maintenance (repérage : classements type "agences web" + case studies e-commerce ; celles qui publient déjà du contenu RGAA — signal d'intention (agence-ep, koredge, pandiweb repérées dans l'étude).
5. **Agences spécialisées secteur public local** (marchés mairies/interco — obligation historique + schéma pluriannuel à maintenir).
6. **Studios Shopify/PrestaShop FR** (leurs clients e-commerce > 2 M€ = obligés EAA).
7. **Cabinet d'audit a11y de taille moyenne** (pas Access42 direct : un challenger) — angle partenariat : ton monitoring entre leurs audits annuels.
8. **Freelances auditeurs RGAA** (LinkedIn #RGAA, communauté a11y FR) — prescripteurs + sous-traitants de ta checklist humaine.
9. **Plateformes SaaS FR e-commerce obligées** (banque en ligne/billetterie de taille moyenne) en direct — pour tester le prix « fin de marché ».
10. **Syndic/collectif d'agences** (France Digitale, agences groupées type collectif d'indés) — 1 deal = N agences.

### 6.2 Angle & script

Angle : **« Carrefour a été condamné en juin. Vos clients e-commerce vous poseront la question — vous répondez quoi ? »** — tu vends à l'agence un produit qu'elle revend avec marge, pas une contrainte.

> Objet : la question RGAA que vos clients vont vous poser
>
> Bonjour [Prénom], depuis la condamnation de Carrefour (juin 2026, accessibilité), les e-commerçants > 2 M€ commencent à interroger leurs agences.
> J'ai construit un monitoring RGAA pensé pour les agences : scan continu de tous vos sites clients, alertes de régression, rapports à votre marque — que vous revendez.
> Honnêteté : l'automatique couvre ~35 % des critères ; on le dit dans chaque rapport, et ça n'a jamais empêché un client de payer, au contraire.
> 30 min pour scanner 3 de vos sites en live ? Le scan d'essai est gratuit : [lien].

### 6.3 Offre pilote, prix test

**Pilote agence** : 3 mois à **249 €/mois pour 15 sites** (au lieu de 399), rapport white-label inclus + 1 checklist humaine offerte sur le site de leur choix — engagement : un retour visio/mois. Self-serve ensuite : **99 € (5 sites) / 249 € (15) / 499 €/mois (40)**. Add-on revue humaine : 149 €/site.

### 6.4 Validé si (45 jours — cycle agence plus lent que devtool)

≥ 3 agences pilotes payées OU 1 pilote + 1 cabinet partenaire signé · ≥ 5 calls où l'agence confirme des demandes clients entrantes RGAA (sinon la douleur est théorique) · scan gratuit : ≥ 100 utilisations, ≥ 15 % emails.
**Invalidé si** : les agences répondent « on enverra chez un cabinet le jour où on nous le demande » (pas de récurrence perçue) · 0 paiement après 15 démos · les prospects ne veulent payer qu'à l'audit one-shot → pivot : outil B2B pour cabinets d'audit, ou retour FacturX/Facturia.
