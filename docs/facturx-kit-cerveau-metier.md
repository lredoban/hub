# FacturX Kit — Cerveau métier, spec moteur, tests, MVP, moat, validation

*7 juillet 2026 · basé sur : Factur-X 1.09 (FNFE-MPE, juin 2026, = ZUGFeRD 2.5), EN 16931-1:2025, Schematron officiel ConnectingEurope 1.3.16 (avril 2026), spécifications externes DGFiP v3.0 + v3.1 (oct. 2025), LF 2026 (sanctions). ⚠️ Ces versions bougent tous les 3-6 mois — c'est précisément ta source de revenu récurrent (§5).*

---

## 1. Carte complète du domaine

### 1.1 Concepts métier

| Concept | Définition opérationnelle | Piège |
|---|---|---|
| **Facture électronique** (sens fiscal 2026) | Fichier structuré (Factur-X, UBL ou CII) transmis via une plateforme agréée. Un PDF simple envoyé par email **n'est plus une facture** en B2B domestique à partir de sept. 2026/2027. | Beaucoup de prospects croient qu'un "PDF propre" suffit. C'est ton hook commercial. |
| **Factur-X** | PDF/A-3 contenant un XML CII nommé exactement `factur-x.xml` + métadonnées XMP (`fx:ConformanceLevel`, `fx:DocumentType=INVOICE`, `fx:DocumentFileName`, `fx:Version`). Le PDF est lisible par l'humain, le XML par la machine. | Le XMP doit **matcher** le GuidelineID du XML. Incohérence = rejet. Relation d'attachement `Data` ou `Alternative`. |
| **Profils Factur-X** (5) | MINIMUM → BASIC WL → BASIC → EN 16931 → EXTENDED. Le profil est déclaré dans BT-24 (GuidelineID, URN). | MINIMUM et BASIC WL ne portent pas les lignes de facture : ce ne sont **pas** des factures structurées complètes au sens fiscal — le socle réforme attend BASIC minimum. Ne jamais laisser un client croire que MINIMUM le met en règle. |
| **EN 16931** | Modèle sémantique européen : ~160 termes BT-x (Business Terms) groupés en BG-x, décliné en 2 syntaxes (CII UN/CEFACT, UBL OASIS). Révision EN 16931-1:2025 approuvée nov. 2025. | Un même BT n'a pas le même XPath en CII et en UBL. Ton modèle interne doit être sémantique (BT), pas syntaxique. |
| **Règles de validation** | ~955 règles : BR-\* (obligations), BR-CO-\* (cohérence calculs), BR-S/Z/E/AE/G/O/IC/K-\* (par catégorie TVA UNTDID 5305), + codes (UNTDID 1001, 4461, VATEX…). Artefacts officiels : Schematron ConnectingEurope (EUPL 1.2, réutilisable). | Ne réécris PAS les 955 règles à la main : exécute le Schematron officiel, et ajoute ta couche au-dessus. |
| **Cycle de vie** | Statuts normalisés (spécs externes v3.1) : déposée, émise, reçue, mise à disposition, prise en charge, approuvée, **rejetée** (non-conformité technique), **refusée** (désaccord commercial), paiement transmis, encaissée. | Rejetée ≠ refusée. Tes messages d'erreur doivent aider à éviter la première ; tu n'as rien à dire sur la seconde. |
| **E-reporting** | Transmission à la DGFiP des données de transactions hors e-invoicing (B2C, international) + données de paiement (pour les prestations de services). | Hors périmètre produit V1 (réservé aux flux PDP), mais dans le périmètre **conseil/contenu** — les prospects confondent tout. |
| **PDP / PA, PPF, annuaire** | PDP (≈138 immatriculées) = transport, cycle de vie, e-reporting. PPF réduit (oct. 2024) à annuaire (SIREN→PDP) + concentrateur fiscal. | **Tu n'es pas une PDP et tu ne le seras jamais** (immatriculation lourde). Positionnement : la brique format/qualité EN AMONT de n'importe quelle PDP. |
| **Types de document** | UNTDID 1001 : 380 facture, 381 avoir, 384 facture rectificative, 386 acompte, 389 auto-facturation. | Un avoir en EN 16931 a des montants **positifs** avec TypeCode 381 (pas des montants négatifs sur un 380) — erreur classique n°1 des devs. |
| **Mentions obligatoires FR** | Art. 242 nonies A ann. II CGI + L441-9 c. com. + **4 nouvelles mentions** (décret 2022-1299, applicables au rythme de l'obligation d'émission) : SIREN du client, adresse de livraison si ≠ facturation, nature de l'opération (biens / services / mixte), mention « option pour le paiement de la TVA d'après les débits » le cas échéant. | La conformité EN 16931 ne garantit PAS la conformité aux mentions françaises. C'est ta couche différenciante `rules-fr`. |

### 1.2 Acteurs

- **Ton client direct** : le dev/CTO d'un **éditeur SaaS FR** qui facture pour ses utilisateurs (facturation, gestion, verticaux métier, e-commerce) ; le **freelance intégrateur** (Odoo, ERP, PrestaShop) qui équipe SES clients.
- **Utilisateur final** (indirect) : le fournisseur émetteur (TPE/PME), son acheteur, son expert-comptable — tu ne leur parles jamais directement, mais tes messages d'erreur finiront sous leurs yeux.
- **Systèmes** : la PDP choisie par l'utilisateur final (destinataire de tes fichiers), la DGFiP (via PDP), le validateur FNFE-MPE (juge de paix officieux).

### 1.3 Données nécessaires pour une facture EN 16931 (checklist d'entrée)

Vendeur : raison sociale, adresse complète + code pays, **SIREN/SIRET** (BT-30/BT-29 schème 0002/0009), n° TVA intracom (BT-31), forme juridique + capital (mention FR), RCS.
Acheteur : raison sociale, adresse + pays, **SIREN** (nouvelle mention 2026), TVA intracom si autoliquidation/intracom.
Document : numéro (séquentiel, sans trou — règle FR), date d'émission, TypeCode, devise (BT-5), **date de livraison/fin de prestation** (BT-72), période le cas échéant, référence bon de commande (BT-13), adresse de livraison si ≠ (BG-15).
Lignes : dénomination, quantité + unité (UN/ECE Rec 20), prix unitaire net, remises ligne, **catégorie + taux TVA par ligne** (BT-151/152).
Pied : remises/charges globales (BG-20/21), ventilation TVA par taux (BG-23 : base, taux, montant), totaux BT-106→115, TVA en devise nationale (BT-111) si facture en devise étrangère.
Paiement : moyen (UNTDID 4461), IBAN, date d'échéance (BT-9) ou conditions (BT-20), **pénalités de retard + indemnité forfaitaire 40 € + conditions d'escompte** (mentions FR, en texte BT-20/BT-22).

### 1.4 Règles à encoder en priorité (au-delà du Schematron officiel)

1. **Arithmétique** (BR-CO-10/13/14/15/17) : Σ lignes = BT-106 ; BT-109 = BT-106 − BT-107 + BT-108 ; BT-110 = Σ montants TVA par catégorie ; TVA par catégorie = arrondi(base × taux) **par groupe de taux, pas par ligne** ; TTC = HT + TVA. Arrondi half-up à 2 décimales sur les montants ; prix unitaires libres (4+ décimales acceptées).
2. **Cohérence TVA** : catégorie S exige taux > 0 ; Z exige 0 ; E/AE/G/O exigent taux 0 ou absent **+ motif d'exonération** (BT-121 code VATEX ou BT-120 texte). Taux FR admis : 20, 10, 5.5, 2.1 (+ DOM). Franchise en base → catégorie E + « TVA non applicable, art. 293 B du CGI ».
3. **Mentions FR** : les 4 nouvelles 2026 + pénalités/40 €/escompte + numérotation séquentielle. Sanction : **50 €/facture non conforme, plafond 15 000 €/an** (LF 2026) — chiffre à mettre dans tes messages et ta landing.
4. **Container PDF** : PDF/A-3 valide, pas de JS, polices embarquées, XML `factur-x.xml`, XMP cohérent.
5. **Profil déclaré vs contenu réel** : un XML déclaré EN 16931 qui contient des champs EXTENDED = invalide ; déclaré BASIC sans lignes = invalide.

### 1.5 Exceptions et cas limites (les vrais, ceux qui font les tickets support)

Avoirs (381) et rectificatives (384) avec référence à la facture d'origine (BG-3) · acomptes (386) puis facture finale avec déduction (BT-113 montant déjà payé) · autoliquidation sous-traitance BTP (AE + « Autoliquidation ») · ventes intracom (catégorie K/IC, TVA 0, deux n° TVA requis) · export hors UE (G) · facture en USD avec BT-111 en EUR · remise globale + charge globale simultanées · ligne gratuite (prix 0 autorisé, quantité négative interdite → avoir) · quantité fractionnaire (0.33 h) et arrondis qui divergent ligne/pied · multi-taux (20 + 10 + 5.5 sur une facture) · auto-facturation (389, mention « Autofacturation ») · factures de situation BTP (EXTENDED uniquement) · vendeur micro-entrepreneur (franchise 293 B) · acheteur public (rester sur Chorus Pro — hors périmètre, à détecter et le dire).

### 1.6 Zones d'incertitude (à surveiller, pas à trancher)

- Détail exact des codes statuts v3.1 et exigences d'interopérabilité PDP (tests en cours 2026).
- Date d'adoption par la France de EN 16931-1:2025 (vs :2017+A1) dans les contrôles PDP.
- Rythme Factur-X : 1.07.3 → 1.08 (01/2026) → 1.09 (06/2026). Quelle version chaque PDP accepte = matrice mouvante → **c'est un contenu payant** (table de compatibilité maintenue).
- Passage éventuel du seuil MINIMUM/BASIC dans les faits (certaines PDP acceptent, d'autres non).

### 1.7 À ne JAMAIS promettre (légal)

1. « Conformité garantie » ou « votre facture sera acceptée » — tu vérifies des règles publiées, la décision finale appartient à la PDP réceptrice et à l'administration.
2. Du **conseil fiscal** (choix d'un taux, d'un régime, d'une option TVA) — toujours « vérifiez avec votre expert-comptable », sinon exercice illégal + responsabilité.
3. L'**archivage à valeur probante** (10 ans) ou la piste d'audit fiable — hors produit V1.
4. La **transmission** à la DGFiP / e-reporting — réservé aux PDP immatriculées ; tu es un outil de préparation/qualité.
5. Toute validation PDF/A « certifiée ISO 19005 » si tu n'exécutes pas veraPDF — dis exactement ce que tu vérifies.
→ Traduire en CGV : obligation de moyens, pas de résultat ; l'utilisateur reste l'émetteur légal de ses factures.

---

## 2. Spécification fonctionnelle du moteur

### 2.1 Modules (monorepo)

```
@facturx-kit/core      MIT  — modèle sémantique (BT/BG typés), calculs, build/parse CII
@facturx-kit/pdf       MIT  — embed/extract factur-x.xml dans PDF/A-3, XMP
@facturx-kit/validate  MIT (moteur) — orchestration XSD + Schematron officiel + règles maison
@facturx-kit/rules-fr  source-available — mentions FR, sanctions, franchise, autoliquidation
@facturx-kit/cli       MIT  — `facturx validate f.pdf`, `facturx generate invoice.json`
api/                   fermé — API hébergée + validateur web + billing
```

### 2.2 Inputs / Outputs

**generate** — input : `InvoiceInput` (JSON validé par zod ; montants en **centimes entiers** ou Decimal — jamais de float) + options `{profile, output: 'xml'|'pdf', basePdf?}`. Output : `factur-x.xml` seul, ou PDF/A-3 complet (à partir du PDF du client via `basePdf`, ou du template PDF minimal intégré). Le moteur **calcule lui-même** les totaux et la ventilation TVA à partir des lignes — l'appelant ne fournit jamais un total (source n°1 d'incohérences) ; s'il en fournit un, on le vérifie et on refuse s'il diverge.

**validate** — input : PDF, XML, ou les deux ; option `{level: 'schema'|'en16931'|'fr', profileExpected?}`. Output :

```ts
{
  verdict: 'PASS' | 'PASS_WITH_WARNINGS' | 'FAIL' | 'INDETERMINATE',
  profile: { declared: 'EN16931', detected: 'EN16931', match: true },
  score: { errors: 2, warnings: 3, checksRun: 214, checksSkipped: ['PDFA_FULL'] },
  findings: [{
    code: 'BR-CO-13',            // ou FRX-PDF-003, FR-MENT-02…
    severity: 'error'|'warning'|'info',
    bt: 'BT-109', xpath: '…SpecifiedTradeSettlementHeaderMonetarySummation…',
    message: "Total HT (1 000,00 €) ≠ somme des lignes − remises + charges (1 010,00 €).",
    fix: "Écart de 10,00 € : vérifiez la remise globale BG-20 — elle est déclarée mais non déduite du total.",
    ref: "https://…/rules/BR-CO-13"
  }],
  disclaimer: "Contrôle des règles publiées (EN 16931 1.3.16, Factur-X 1.09, mentions CGI). Ne constitue ni conseil fiscal ni garantie d'acceptation."
}
```

**extract** — PDF/XML → `InvoiceInput` JSON (round-trip garanti avec generate). **convert** (V1.1) — CII ↔ UBL.

### 2.3 Décisions du moteur (pipeline)

1. **Sniffing** : PDF ? → extraire `factur-x.xml` (absent → FAIL FRX-PDF-001 « ce PDF ne contient pas de facture structurée : ce n'est pas une facture électronique au sens de la réforme »). XML ? → CII ou UBL (namespace).
2. **Profil** : lire BT-24 ; inconnu → FAIL ; MINIMUM/BASIC WL → **warning systématique** « profil insuffisant comme facture fiscale structurée ».
3. **Couches** : XSD (bloquant) → Schematron EN 16931 officiel (erreurs/warnings selon flag) → règles arithmétiques maison (re-calcul indépendant en Decimal) → `rules-fr` (mentions) → contrôles PDF (XMP, nom de pièce jointe, relation) .
4. **Niveaux de confiance** : `PASS` = 0 erreur, 0 warning. `PASS_WITH_WARNINGS` = 0 erreur. `FAIL` = ≥1 erreur. `INDETERMINATE` = le moteur **refuse de conclure** : PDF chiffré, XML tronqué, profil EXTENDED avec extensions sectorielles non couvertes, PDF/A complet non vérifiable (on liste `checksSkipped`). Ne jamais transformer un INDETERMINATE en PASS.
5. **Explications** : chaque finding = message FR orienté action (quoi, où, combien, comment corriger), jamais le texte brut du Schematron. Mapping `BR-* → message humain` = ton actif propriétaire (§5).

### 2.4 API hébergée (V1)

```
POST /v1/validate        multipart (file) | JSON {xml}     → rapport (cf. 2.2)
POST /v1/generate        JSON InvoiceInput                  → XML ou PDF (base64/binaire)
POST /v1/extract         multipart (file)                   → InvoiceInput JSON
GET  /v1/health, GET /v1/versions   (versions des règles embarquées — argument de vente)
Auth: Bearer clé API. Erreurs HTTP: 400 input illisible, 402 quota, 422 rapport FAIL (avec body complet), 200 PASS/WARN.
```
**Stateless par défaut : aucun stockage des factures** (ni fichier ni contenu, purge immédiate, logs = métadonnées seulement). C'est un argument commercial (RGPD/secret des affaires), pas une limite.

---

## 3. Corpus de tests (55 scénarios)

Format : `ID · description · attendu`. Chaque scénario = un fixture JSON/XML/PDF dans `packages/core/fixtures/`. Les IDs sont stables (ils deviendront ta suite de conformité publique, §5).

**Génération — cas simples (G)**
1. G-01 · Facture 1 ligne, TVA 20 %, B2B FR complet · XML EN 16931 PASS au validateur FNFE + Schematron 1.3.16.
2. G-02 · 3 lignes même taux · BT-106 = Σ lignes ; une seule ligne BG-23.
3. G-03 · Multi-taux 20/10/5.5 · 3 groupes BG-23, BT-110 = Σ des 3 arrondis.
4. G-04 · Remise globale 50 € (BG-20) · BT-107 = 50, BR-CO-13 vérifié.
5. G-05 · Charge globale (frais de port taxés 20 %) · BG-21 dans la base TVA du bon groupe.
6. G-06 · Avoir complet (381) référençant G-01 · montants positifs, BG-3 rempli.
7. G-07 · Génération PDF sans basePdf (template minimal) · PDF/A-3 + XMP cohérent + factur-x.xml.
8. G-08 · Embed dans basePdf client (PDF classique) · conversion A-3, avertissement si polices non embarquées.
9. G-09 · Profil BASIC demandé · GuidelineID BASIC, champs EN16931-only omis.
10. G-10 · Round-trip : generate → extract → deep-equal InvoiceInput.

**Calculs & arrondis (C)**
11. C-01 · 3 × 0,333 € TVA 20 % · arrondi au groupe TVA : base 1,00 → TVA 0,20 ; jamais Σ d'arrondis de lignes.
12. C-02 · Quantité 0,33 h × 150 €/h · prix unitaire 4 décimales accepté, montants 2 décimales.
13. C-03 · Cas limite half-up : base 10,125 € × 20 % = 2,025 → 2,03 (documenter la règle choisie et s'y tenir).
14. C-04 · Remise ligne 100 % (ligne gratuite) · prix net 0 valide, base TVA 0.
15. C-05 · Facture 0 € (full remise) · valide EN 16931 + warning « facture à 0 € : vérifiez l'intérêt fiscal ».
16. C-06 · Totaux fournis par l'appelant divergents des lignes · generate REFUSE (erreur E-CALC-001), ne « corrige » jamais silencieusement.
17. C-07 · 10 000 lignes · perf < 2 s, pas de dépassement mémoire.
18. C-08 · Montants ≥ 10^9 · pas de notation scientifique dans le XML.
19. C-09 · Acompte : facture finale avec BT-113 = 300 · BT-115 (à payer) = TTC − 300.
20. C-10 · BT-114 arrondi total ±0,01 · accepté et répercuté dans BT-115.

**TVA & régimes (T)**
21. T-01 · Franchise en base (293 B) · catégorie E, taux 0, BT-120 « TVA non applicable, art. 293 B du CGI » exigé — sinon FAIL FR-VAT-01.
22. T-02 · Autoliquidation sous-traitance BTP · catégorie AE + n° TVA des deux parties + mention « Autoliquidation » — 3 checks distincts.
23. T-03 · Livraison intracom · catégorie K, TVA acheteur obligatoire, mention 262 ter I.
24. T-04 · Export hors UE · catégorie G + VATEX.
25. T-05 · Catégorie S avec taux 0 · FAIL BR-S-05-like (incohérence catégorie/taux).
26. T-06 · Taux 19,6 % (périmé) · FAIL FR-VAT-02 « taux inconnu en France (admis : 20, 10, 5.5, 2.1) ».
27. T-07 · Exonération sans motif (ni BT-120 ni BT-121) · FAIL BR-E-10-like.
28. T-08 · Mix taxable + exonéré sur une facture · 2 groupes BG-23 dont un E avec motif.
29. T-09 · TVA sur les débits (option) · mention présente → info OK ; nature services sans mention → warning FR-MENT-05.
30. T-10 · Facture USD · BT-5 = USD, BT-111 (TVA en EUR) obligatoire → FAIL si absent.

**Mentions FR 2026 (F)**
31. F-01 · SIREN client absent (B2B FR) · FAIL FR-MENT-01 avec le chiffre « 50 €/facture, plafond 15 000 €/an ».
32. F-02 · Adresse de livraison ≠ facturation non renseignée alors que ship-to connu · warning FR-MENT-02.
33. F-03 · Nature de l'opération absente · FAIL FR-MENT-03 (biens/services/mixte).
34. F-04 · Pénalités de retard / indemnité 40 € absentes · warning FR-MENT-04 (L441-9) — warning car sanction civile distincte.
35. F-05 · Numérotation avec trou détectable (série fournie) · warning FR-SEQ-01.
36. F-06 · SIREN à 8 chiffres (typo) · FAIL format + checksum Luhn.
37. F-07 · N° TVA FR incohérent avec SIREN (clé) · FAIL FR-VAT-03 (clé TVA = f(SIREN) recalculée).
38. F-08 · Vendeur DOM (TVA 8,5/2,1) · accepté, hors scope warning doc.

**Parsing / PDF (D & X)**
39. D-01 · PDF sans XML embarqué · FAIL FRX-PDF-001 (message pédagogique réforme).
40. D-02 · XML nommé `zugferd-invoice.xml` (ZUGFeRD 1.x legacy) · détecté, warning « renommer factur-x.xml ».
41. D-03 · XMP ConformanceLevel=BASIC mais XML EN16931 · FAIL FRX-PDF-002 (incohérence profil).
42. D-04 · PDF chiffré par mot de passe · INDETERMINATE (refus de conclure).
43. D-05 · PDF non-A/3 (PDF 1.4 simple) avec XML valide · FAIL FRX-PDF-003 + fix « régénérer le container ».
44. D-06 · 2 pièces jointes XML · FAIL (ambiguïté).
45. X-01 · XML UBL en entrée du validateur · V1 : INDETERMINATE explicite « UBL non couvert, prévu Q4 » (jamais un faux FAIL).
46. X-02 · XML tronqué/mal formé · FAIL E-SCHEMA-001 ligne/colonne.
47. X-03 · Encodage ISO-8859-1 déclaré UTF-8 · FAIL avec détection d'encodage.
48. X-04 · XML 60 Mo · rejet 413 au-delà de la limite documentée (10 Mo), pas de crash.

**Invalides, ambigus, refus de conclure (I & A)**
49. I-01 · TypeCode 380 avec montants négatifs · FAIL + fix « émettre un avoir 381 ».
50. I-02 · Date d'échéance antérieure à l'émission · warning (légal mais louche).
51. I-03 · Injection XXE / entités externes dans le XML · parseur durci, entités refusées, test sécurité bloquant.
52. I-04 · GuidelineID EXTENDED avec données sectorielles · PASS des règles communes + INDETERMINATE sur l'extension (périmètre dit).
53. A-01 · Acheteur = entité publique (SIRET connu comme administration) · warning « circuit Chorus Pro, hors périmètre B2B » — on ne conclut pas.
54. A-02 · Question implicite de régime (auto-entrepreneur qui demande « quel taux ? ») via des données contradictoires (293 B + taux 20 sur une ligne) · FAIL cohérence + message « choix de régime = expert-comptable », jamais de recommandation fiscale.
55. A-03 · Facture déclarée conforme « spécifications v3.0 » alors que le moteur embarque v3.1 · PASS_WITH_WARNINGS + note de version (transparence sur le référentiel utilisé).

**Definition of done tests** : les 55 fixtures passent ; G-01→G-06 passent AUSSI le validateur FNFE-MPE et Mustang (validation croisée automatisée en CI) ; mutation testing sur le module de calcul.

---

## 4. MVP vendable

### 4.1 Périmètre V1 (indispensable)

1. `core` : InvoiceInput zod → **génération CII EN 16931** (profils BASIC + EN 16931 seulement) avec calculs internes.
2. `pdf` : embed/extract PDF/A-3 + XMP (via pdf-lib ; conversion A-3 basique documentée « best effort »).
3. `validate` : XSD + Schematron officiel 1.3.16 (compilé en XSLT, exécuté via saxon-js) + 20 règles FR/calc maison prioritaires (celles du corpus).
4. CLI (`npx facturx validate facture.pdf`) — c'est ta démo commerciale.
5. API hébergée `/validate` + `/generate` + clés + quotas + Stripe.
6. **Validateur web gratuit** (upload → rapport HTML) = machine à leads.
7. Docs (Docus/VitePress) : quickstart 5 min, table des règles, guide réforme.

### 4.2 Simulable manuellement au début

Rapport « revue expert » PDF (toi + 1 h) vendu 149 € · onboarding pilote (visio) · conversion UBL à la main si un pilote l'exige · veille normative envoyée par email (avant d'être un flux produit) · matrice de compatibilité PDP (Notion partagé).

### 4.3 Inutile avant les premiers paiements

Dashboard SaaS, orgs/teams, UBL automatique, EXTENDED, e-reporting, connecteurs PDP, éditeur visuel de facture, archivage probant, SSO, i18n anglais, app Shopify (c'est Facturia, plus tard, SUR cette lib).

### 4.4 Architecture minimaliste

```
Monorepo pnpm + TS strict + vitest.
API: Hono sur Node 22 (1 conteneur Railway), stateless.
DB: Postgres Railway (4 tables) — AUCUNE donnée de facture stockée.
Billing: Stripe Checkout + webhooks (customer.subscription.*, usage-based simple).
Site + docs + validateur web: Nuxt statique (tu es chez toi) → Railway/Pages.
Observabilité: Sentry + logs JSON (jamais le contenu des factures).
```

**Modèle de données** :
```sql
users(id, email, stripe_customer_id, created_at)
api_keys(id, user_id, key_hash, label, created_at, revoked_at)
subscriptions(id, user_id, stripe_sub_id, plan, quota_monthly, status)
usage_events(id, api_key_id, endpoint, verdict, duration_ms, created_at)  -- métadonnées only
```

### 4.5 Ordre exact de build (~15 jours de code)

J1-2 `core` modèle BT + calculs + fixtures C-\* → J3-4 build CII + G-01→G-06 validés contre FNFE → J5 `pdf` embed/extract (D-\*) → J6-7 `validate` pipeline Schematron + messages FR des 30 findings les plus fréquents → J8 CLI + publication npm v0.1 + README comparatif → J9-10 API Hono + clés + quotas + Stripe → J11 validateur web gratuit branché sur l'API → J12 docs + page règles → J13 CI conformité croisée (FNFE/Mustang) → J14-15 buffer + landing branchée sur la vraie API (remplace la fake door).
Règle : **npm v0.1 publié à J8 même imparfait** — l'outreach du plan 14 jours (comité) a besoin d'un lien qui marche.

---

## 5. Moat solo-dev

| Couche | Statut | Pourquoi |
|---|---|---|
| `core`, `pdf`, CLI, moteur validate | **MIT** | Acquisition. Devenir la lib TS par défaut = distribution gratuite permanente. |
| Schematron officiel | EUPL (déjà public) | Aucune valeur à cacher — la valeur est dans l'exécution + messages. |
| `rules-fr` + messages actionnables + fix hints | **source-available** (licence non-concurrence) ou API-only | C'est le savoir métier FR, coûteux à reconstituer. |
| API hébergée, rapports blancs, suite de conformité continue, matrice PDP | **Payant** | Récurrent. |

**Ce qui devient difficile à copier** : (1) le **mapping 955 règles → messages français actionnables avec fix** (des mois de travail incrémental nourri par les tickets) ; (2) le **corpus de fixtures réelles** accumulé via le validateur gratuit (chaque upload anonymisé en erreur = un cas de test de plus) ; (3) la **fraîcheur** : être à jour ≤ 2 semaines après chaque release Factur-X/spécs externes — un concurrent doit maintenir, pas juste copier.

**Effet « source de vérité »** : publie la suite de tests de conformité en OSS (`facturx-conformance`, les 55 IDs ci-dessus) et invite les autres libs à s'y mesurer ; badge « FacturX Kit conformance N.N » ; changelog normatif public (« ce qui change dans Factur-X 1.09 pour votre code ») qui devient LA page que les devs FR consultent à chaque release ; page de compatibilité PDP maintenue. Celui qui écrit les tests écrit le standard de facto.

**Crédibilité petite structure** : adhésion FNFE-MPE (affichable) · validation croisée automatique vs validateur FNFE et Mustang publiée en CI (« nos 55 cas passent les 3 validateurs ») · zéro stockage des factures (argument sécurité qu'un gros SaaS ne peut pas donner) · status page + versions des règles exposées dans l'API (`/v1/versions`) · ton vrai nom et ton historique Kompa sur la page About — en France, en 2026, « le dev qui a déjà implémenté Factur-X en prod » est plus crédible qu'un logo.

---

## 6. Plan de validation commerciale

### 6.1 Dix prospects (nominatifs, à requalifier en 30 min sur LinkedIn)

1. **Obat** (logiciel devis/factures BTP) — l'autoliquidation BTP est leur cauchemar → cas T-02.
2. **Tolteck** (devis/factures artisans) — même segment, plus petit, décision rapide.
3. **Freebe** (facturation freelances) — franchise 293 B massive chez leurs users → T-01.
4. **Abby** (compta/facturation micro-entrepreneurs) — idem + croissance forte.
5. **Henrri / Rivalis** (facturation gratuite PME) — gros volume, pas une PDP à ma connaissance.
6. **Facture.net (Codeur.com)** — outil gratuit à moderniser, la conformité est leur seule feature à vendre.
7. **Éditeurs de modules PrestaShop FR** (ex. 202 ecommerce, agences top du marketplace Addons) — 1 module Factur-X = des centaines de marchands.
8. **Intégrateurs Odoo FR de taille moyenne** (hors Akretion) — Malt/annuaire Odoo, ils facturent l'intégration, ta lib leur fait gagner des jours.
9. **SaaS verticaux avec facturation intégrée** (résa ateliers, coachs, agences — ex. les concurrents d'Izidoor/Wecandoo croisés dans tes recherches) — ils découvrent à peine qu'ils sont concernés.
10. **Ton propre réseau Kompa** : 1 éditeur/partenaire qui a le sujet Factur-X dans son backlog Q3 — le pilote le plus rapide à signer.

### 6.2 Angle & script

Angle : pas « j'ai un outil », mais **« l'échéance du 1er septembre + l'amende 50 €/facture, où en est votre backlog ? »**. Email/DM (≤ 90 mots) :

> Objet : Factur-X dans [Produit] avant le 1er septembre ?
>
> Bonjour [Prénom], je suis dev, j'ai implémenté Factur-X en production pour [contexte Kompa].
> À partir du 1/09/2026 vos utilisateurs devront recevoir (puis émettre) des factures au format structuré — et l'amende est passée à 50 €/facture non conforme.
> J'ai sorti une lib TypeScript open source qui génère/valide du Factur-X EN 16931 (+ mentions françaises 2026) : [lien npm].
> Si le sujet est dans votre backlog, je prends 30 min pour vous montrer comment l'intégrer dans [Produit]. Sinon, gardez la lib, elle est MIT.
> [Prénom]

Le « sinon gardez la lib » est l'arme : taux de réponse dev-to-dev, zéro pression, et chaque non devient un utilisateur OSS.

### 6.3 Offre pilote, prix test

**Pilote « Conforme avant le 1er septembre »** (5 places) : intégration done-with-you (audit du flux de facturation, intégration lib/API, fixtures de LEURS cas réels, 30 j de support direct) — **1 490 € HT + 12 mois d'API Pro inclus**, puis 99 €/mois. API self-serve : **29 / 99 / 299 €/mois** (1k / 10k / 100k validations), early-bird −50 % un an pour les 20 premiers. Revue expert one-shot : 149 € (le produit d'appel qui se vend en un email).

### 6.4 Projet validé si (30 jours)

- ≥ **3 pilotes payés** (4 470 €) OU ≥ 10 abonnements API réels ;
- ≥ 4 des 10 prospects disent en call « on doit régler ça avant septembre » (H1/H3 du comité confirmées) ;
- validateur web : ≥ 200 fichiers analysés et ≥ 10 % laissent leur email ;
- npm : ≥ 500 dl/semaine en tendance après 3 semaines.

**Invalidé si** : ≤ 1 paiement malgré 10 calls · ≥ 6/10 prospects répondent « notre PDP gère le format pour nous » · le validateur gratuit tourne mais 0 conversion payante → la valeur perçue est dans le transport (PDP), pas le format → pivot : app Shopify (Facturia) sur la même lib, ou Veillance.
