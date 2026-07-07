# Veillo — Cerveau métier, spec moteur, tests, MVP, moat, validation

*7 juillet 2026 · Fiches de garde interactives (plantes, animaux, maison) partagées par lien sans compte. Contexte : WTP faible et usage saisonnier (2-3×/an) identifiés dès la grille Notion ; l'angle pro (Hunimalis, marketplaces petsitting) est pris. Produit à boucle virale + monétisation one-shot légère — PAS un SaaS d'abonnement. Le comité l'a exclu de la shortlist : ce document existe pour cadrer un éventuel build « été » à côté de FacturX Kit, pas pour le remettre en pari principal.*

---

## 1. Carte complète du domaine

### 1.1 Concepts métier

| Concept | Définition opérationnelle | Piège |
|---|---|---|
| **Fiche de garde** | Document vivant qui transfère la charge mentale du propriétaire vers le gardien : sujets (1 chat, 12 plantes, la maison), consignes, rythmes, urgences. | Ce n'est PAS un doc statique (sinon WhatsApp/Google Doc gagne) : la valeur = le **mode garde** exécutable jour par jour. |
| **Sujet de garde** | Entité gardée : animal (espèce, tempérament, véto), plante (emplacement, arrosage), maison (courrier, poubelles, alarme, volets). | Le modèle doit être polymorphe dès le départ — les gardes réelles mélangent toujours les trois. |
| **Consigne** | Instruction atomique : texte + photo + fréquence (quotidien matin/soir, tous les X jours, date fixe, « si besoin ») + criticité (vital / important / confort). | La criticité pilote tout : rappels, ordre d'affichage, ton des notifications. Nourrir le chat = vital ; arroser le ficus = confort. |
| **Période de garde** | Dates + gardien(s) affectés. Génère le **planning** : consignes × jours = occurrences cochables. | Fuseaux et « jour 1 » : la garde commence souvent un soir. Générer les occurrences en heure locale du foyer, pas UTC. |
| **Lien d'accès invité** | URL + token : le gardien n'installe rien, ne crée pas de compte. Optionnel : code PIN. | LE différenciateur UX et LE risque sécurité (voir 1.7). |
| **Journal de garde** | Coches, photos, notes du gardien (« Miso a bien mangé » + photo). | C'est la **boucle émotionnelle** qui fait revenir le propriétaire (et le convertit) : le produit se vend au moment où le proprio reçoit la photo du chat. |
| **Modèle / template** | Fiches types par espèce/contexte (chat, chien, poules, orchidées, piscine…) pré-remplies. | Double usage : onboarding en 3 min ET SEO (« fiche de garde chat à imprimer » etc.). |
| **Contact d'urgence** | Vétérinaire, voisin, plombier, numéro du proprio à l'étranger, n° d'urgence véto de garde. | Affichage hors ligne obligatoire (le gardien panique dans une zone sans réseau). |

### 1.2 Acteurs

- **Propriétaire** (payeur, crée la fiche) : couple 28-45 ans, urbain, chat/plantes, 2-4 départs/an. Persona secondaire : propriétaire de maison secondaire.
- **Gardien** (invité sans compte) : ami/famille/voisin (bénévole, il rend service — l'UX doit le flatter, pas le fliquer) ; ou petsitter semi-pro.
- **Sitter pro / conciergerie** (segment V2) : gère 10-40 foyers, voudrait des fiches par client — c'est LUI le seul abonnement récurrent plausible.
- **Le chat** : partie prenante silencieuse, star des photos, moteur de la viralité.

### 1.3 Données nécessaires

Foyer : adresse (⚠️ sensible), accès (clés, code — ⚠️ très sensible), wifi. Sujets : nom, photo, espèce/type, consignes par criticité, quantités (doses de croquettes — photo du verre doseur > texte), véto + carnet. Garde : dates, gardien(s) (prénom + tel), lien/PIN. Journal : occurrences cochées, photos, notes horodatées.

### 1.4 Règles produit

1. **Zéro compte gardien**, à jamais. Toute friction gardien tue le produit (il rend service, il n'a rien demandé).
2. **Le lien expire** : fin de garde + 7 jours par défaut. Un lien de garde n'est pas un accès permanent au domicile de quelqu'un.
3. **Occurrences générées, jamais recalculées à la volée** : le journal est un fait historique (le gardien a coché à 8h12), même si la consigne change ensuite.
4. **Criticité vitale = redondance** : consigne vitale non cochée à l'heure limite → rappel gardien, puis notification propriétaire (opt-in, formulée sans dramatiser).
5. **Offline-first** côté gardien : la fiche consultée une fois reste lisible sans réseau (PWA cache) ; les coches se synchronisent au retour. (Cohérent avec l'envie Notion de terrain local-first/CRDT — mais un cache PWA suffit au MVP, le CRDT est un luxe.)
6. **Une fiche gratuite, complète, pour toujours** : le paywall porte sur multi-fiches/multi-gardes simultanées + options confort, jamais sur la sécurité du chat (nourrir un animal ne doit jamais être derrière un paywall — éthique ET évidence PR).

### 1.5 Exceptions & cas limites

Garde qui se prolonge (vol annulé) → le gardien lui-même peut demander l'extension du lien · deux gardiens qui se relaient (voisin matin, ami soir) → occurrences par créneau, qui-fait-quoi visible · animal malade pendant la garde → mode « incident » (photo + note épinglée + carte véto en un tap) · consigne « tous les 3 jours » sur garde de 8 jours → J1 ou J3 ? (règle : premier jour plein, configurable) · plantes dehors + canicule → consignes conditionnelles simples (« s'il fait > 30° ») en texte, PAS de moteur météo au MVP · proprio injoignable (avion) → escalade vers contact de secours · gardien qui ne coche rien mais fait tout (le papa qui refuse les apps) → mode impression PDF propre, et le journal reste vide sans culpabilisation · départ dans 2 h, fiche à faire → template + dictée vocale = fiche en 3 min (LE cas réel n°1) · foyer bilingue → fiche partageable en 2 langues (V2, deepL sur texte statique).

### 1.6 Zones d'incertitude

WTP réelle : 9 € one-shot passe-t-il quand WhatsApp est gratuit ? (l'hypothèse à tuer en premier) · fréquence d'usage suffisante pour qu'on se souvienne du produit d'un été à l'autre (rétention annuelle, pas mensuelle) · le segment sitter pro veut-il payer ou vit-il très bien avec ses PDF ? · saisonnalité extrême (juillet-août + Noël) — le produit doit être rentable avec 3 pics/an.

### 1.7 À ne JAMAIS promettre (et risques spécifiques)

1. **Sécurité du domicile** : adresse + dates d'absence + code d'alarme dans une même fiche = kit cambriolage. Ne jamais promettre « coffre-fort » ; MAIS traiter le risque produit : champs « accès » chiffrés, affichés seulement pendant la garde, jamais dans les emails, liens expirants, pas d'indexation (noindex + tokens non devinables), option PIN.
2. **Santé animale** : aucune posologie suggérée, aucun conseil vétérinaire (« demandez à votre vétérinaire » systématique). La fiche transmet ce que le PROPRIO a écrit, point.
3. **Pas de médication humaine** : refuser explicitement le cas d'usage « garde de personne » (grand-mère) — c'est Ancre, c'est HDS, c'est non (message doux qui le dit si détecté).
4. **Fiabilité** : jamais « n'oubliez plus rien » → « tout est au même endroit ». Un rappel peut ne pas partir (téléphone éteint) ; le produit est une aide, la responsabilité reste humaine (CGU).
5. RGPD sobre : données du gardien minimales (prénom, tél optionnel), droit à l'oubli simple, pas de tracking tiers dans les fiches.

---

## 2. Spécification fonctionnelle du moteur

### 2.1 Modules

```
core/       — modèle (Household, Subject, Instruction, CarePeriod, Occurrence, JournalEntry)
scheduler/  — génération des occurrences (fréquences × période, TZ locale), état (todo/done/missed)
share/      — tokens invités, expiration, PIN, permissions (viewer/logger)
journal/    — coches, photos, notes ; flux propriétaire temps réel
notify/     — rappels gardien (push PWA/SMS?), digest proprio, escalade « vital manqué »
templates/  — bibliothèque par espèce/contexte (données + contenu SEO)
billing/    — one-shot Stripe (fiche unique gratuite → Pack illimité)
```

### 2.2 Inputs / Outputs

**Créer une garde** : `{household, subjects[], instructions[], period:{start,end,tz}, sitters[]}` → fiche + planning d'occurrences + lien(s) `veillo.app/g/<token>`.
**Vue gardien** (sans compte) : « Aujourd'hui » = occurrences du jour triées par criticité/créneau, cochables, + onglets Sujets / Urgences / Journal. Poids page < 500 Ko, lisible en 3G, installable PWA.
**Vue proprio** : timeline du journal (photos d'abord), état des vitaux du jour en un coup d'œil, bouton « merci 🧡 » (renvoyé au gardien — boucle émotionnelle dans les deux sens).
**Erreurs** : lien expiré → page douce avec « demander la réactivation au propriétaire » (1 tap) ; conflit de coche offline → dernière écriture gagne + les deux notes conservées (jamais de perte de note).

### 2.3 Décisions du moteur

Occurrence `missed` = non cochée à la fin du créneau → visible mais **sans culpabilisation** (le gardien est bénévole) ; seul `vital missed` déclenche l'escalade. Fusion offline : les coches sont des événements append-only (qui, quand, quoi) — pas d'état écrasable. Expiration : token vérifié serveur à chaque sync, mais le cache local reste lisible (lecture seule) jusqu'à +7 j.

---

## 3. Corpus de tests (50 scénarios)

**Création & templates (C)**
1. C-01 · fiche chat via template en < 3 min (parcours complet) · test E2E chronométré.
2. C-02 · fiche mixte chat + 12 plantes + maison · 3 types de sujets cohabitent.
3. C-03 · photo du verre doseur sur consigne croquettes · upload compressé < 300 Ko, EXIF GPS **supprimé**.
4. C-04 · consigne sans fréquence (« si besoin ») · apparaît dans Sujets, jamais dans le planning quotidien.
5. C-05 · garde créée pour ce soir 18h · occurrences J1 = uniquement créneau soir.
6. C-06 · période 3 semaines, consigne « tous les 3 jours » · occurrences J1, J4, J7… (premier jour plein).
7. C-07 · 0 sujet, juste la maison · valide (arrosage extérieur, courrier).
8. C-08 · duplication d'une garde passée pour de nouvelles dates · consignes reprises, journal non copié.
9. C-09 · suppression d'un sujet avec historique · soft-delete, le journal passé reste intact.
10. C-10 · fiche > 50 consignes · avertissement UX (« votre gardien va fuir ») + regroupement proposé.

**Partage & accès (S)**
11. S-01 · lien ouvert sans compte sur iPhone Safari · fiche complète, ajout écran d'accueil proposé.
12. S-02 · token 128 bits non séquentiel · test d'entropie + 404 générique sur token invalide (pas d'oracle).
13. S-03 · lien expiré (garde finie +8 j) · lecture refusée, bouton « demander réactivation ».
14. S-04 · PIN activé · 3 échecs = délai progressif.
15. S-05 · champs « accès maison » (code alarme) · chiffrés au repos, jamais dans les emails/SMS/logs/Sentry, visibles uniquement pendant la période de garde.
16. S-06 · noindex/nofollow sur toutes les pages /g/ · vérif headers + robots.
17. S-07 · deux gardiens, deux liens · chacun voit qui a coché quoi.
18. S-08 · révocation immédiate d'un lien par le proprio · sync bloquée en < 1 min.
19. S-09 · partage du lien par le gardien à un tiers (re-forward) · risque documenté, journal des accès (device/heure) visible proprio.
20. S-10 · export/suppression RGPD du foyer complet · purge photos incluses, gardiens notifiés.

**Planning & journal (P)**
21. P-01 · coche « croquettes matin » à 8h12 · journal horodaté local (Europe/Paris), visible proprio en < 10 s.
22. P-02 · consigne modifiée en cours de garde · occurrences futures régénérées, passées intactes.
23. P-03 · vital non coché à 12h (limite matin) · rappel gardien ; toujours rien à 14h → notification proprio (formulation neutre testée).
24. P-04 · confort non coché · aucun rappel, aucun drame.
25. P-05 · photo + note du gardien · apparaît dans la timeline proprio, bouton merci renvoyé.
26. P-06 · garde à cheval sur changement d'heure (DST) · pas d'occurrence dupliquée/perdue.
27. P-07 · proprio en Asie (TZ+7) · les heures affichées proprio = heure du foyer, étiquetée.
28. P-08 · prolongation de garde demandée par le gardien · proprio approuve en 1 tap, occurrences étendues.
29. P-09 · relai entre 2 gardiens en milieu de garde · l'app affiche « à partir de jeudi c'est Léa ».
30. P-10 · rien coché pendant 3 jours mais gardien actif (vues) · pas d'escalade panique (activité ≠ coches), digest neutre.

**Offline & sync (O)**
31. O-01 · fiche ouverte une fois puis avion mode · tout lisible, urgences incluses.
32. O-02 · 5 coches + 2 photos offline · synchronisées dans l'ordre au retour réseau.
33. O-03 · coche offline sur les deux téléphones des 2 gardiens (même occurrence) · append-only : 2 événements conservés, affichage « déjà fait par Léa ».
34. O-04 · token expiré pendant l'offline · lecture cache OK, écritures refusées au retour avec message clair.
35. O-05 · stockage local plein / cache purgé par iOS · re-fetch silencieux, pas d'écran blanc.

**Monétisation (M)**
36. M-01 · 1re fiche 100 % gratuite sans carte · aucune consigne bloquée.
37. M-02 · création 2e fiche · paywall Pack 9 € one-shot (illimité) — pas d'abonnement.
38. M-03 · paiement Stripe Checkout Apple Pay · < 60 s mobile.
39. M-04 · remboursement · fiches conservées en lecture, création re-bloquée.
40. M-05 · features premium (PDF joli, 2 langues, multi-gardiens > 2) · flaggées, dégradation gracieuse.

**Invalides, ambigus, refus (X)**
41. X-01 · consigne « donner 2 comprimés de Doliprane à mamie » · détection personne humaine → message doux : « Veillo est conçu pour animaux, plantes et maison — pour l'aide à une personne, voyez [ressources aidants] » ; pas de blocage brutal, pas de stockage santé humaine revendiqué.
42. X-02 · posologie vétérinaire dans une consigne · accepté tel quel (texte du proprio) + micro-disclaimer « consignes rédigées par le propriétaire ».
43. X-03 · upload PDF ordonnance véto · stocké comme pièce jointe opaque, jamais parsé/interprété.
44. X-04 · adresse + « code alarme 1234 » collés dans une consigne texte libre · heuristique → suggérer de déplacer vers le champ Accès chiffré.
45. X-05 · emoji-only / langue non détectée dans les consignes · affiché tel quel (pas de sur-ingénierie NLP).
46. X-06 · dates de garde dans le passé · refus création avec correction proposée.
47. X-07 · 100 photos du chat en un jour · quota soft (50/j) + compression, jamais de perte silencieuse.
48. X-08 · gardien demande « et si le chat vomit ? » · le produit ne répond JAMAIS (pas de chatbot véto) : affiche la carte Urgences du proprio.
49. X-09 · tentative d'accès /g/ par crawler connu · 404 + rate-limit IP.
50. X-10 · compte proprio supprimé pendant une garde active · garde en cours protégée jusqu'à sa fin (le chat d'abord), purge après.

---

## 4. MVP vendable

### 4.1 V1 indispensable (l'été est DANS 3 semaines — fenêtre réelle)

1. Création fiche via 5 templates (chat, chien, plantes, maison, mixte) — mobile-first.
2. Lien invité sans compte + expiration + vue « Aujourd'hui » cochable.
3. Journal photos/notes + timeline proprio.
4. Urgences offline (PWA cache basique).
5. Paywall : 1 fiche gratuite → Pack 9 € one-shot (Stripe Checkout).
6. PDF imprimable propre (le cas « papa refuse les apps » ET l'aimant SEO).

### 4.2 Simulable manuellement

Rappels « vital manqué » : V1 = pas de push, un simple récap email quotidien au proprio (cron) — l'escalade temps réel attendra · templates : 5 faits main, pas de générateur · traduction : à la main si un pilote le demande · support : ton WhatsApp.

### 4.3 Inutile avant les premiers paiements

Apps natives, CRDT/local-first complet, segment sitter pro (V2 si traction), moteur météo, chatbot, comptes gardiens (jamais), notifications SMS payantes, marketplace de sitters (piège absolu : c'est un autre métier).

### 4.4 Architecture minimaliste

```
Nuxt full-stack (PWA) sur Railway — un seul déployable.
Postgres : households, subjects, instructions, care_periods, occurrences, journal_events (append-only),
           share_tokens, users, purchases. Champs « accès » chiffrés applicativement (libsodium).
Photos : S3-compatible + resize à l'upload, EXIF strip.
Stripe Checkout one-shot. Emails : Resend (digest quotidien).
Pas de queue, pas de Redis : cron Nitro + pg suffisent à cette échelle.
```

### 4.5 Ordre exact de build (~12 jours — deadline : avant fin juillet)

J1-2 modèle + création fiche + templates → J3 partage token + vue gardien lecture → J4-5 occurrences + coches + append-only → J6 journal photos + timeline proprio → J7 PWA cache urgences (O-01) → J8 paywall Stripe + PDF → J9 sécurité (S-05 chiffrement, S-06 noindex, X-09) → J10 digest email → J11-12 polish mobile + **ta propre garde comme premier test réel** (le plan Notion de départ : vos vacances de cet été).

---

## 5. Moat solo-dev

Franchise : le moat produit est **faible** (un clone se code en 2 semaines). Les défenses réelles :

- **Boucle virale structurelle** : chaque garde expose le produit à 1-3 gardiens non-utilisateurs, au moment exact où ils constatent que c'est mieux que le pavé WhatsApp. Le gardien de juillet est le proprio d'août. Optimiser LA page invitée est plus important que toute feature.
- **Templates + SEO** : « fiche de garde chat », « instructions pet sitting PDF », « qui arrose mes plantes vacances » — contenu gratuit imprimable qui draine vers l'outil. Le corpus de templates par espèce (poules, aquarium, gecko…) devient l'actif de longue traîne.
- **Confiance/sécurité comme marque** : « le seul outil de garde qui chiffre vos codes d'accès et expire les liens » — argument que WhatsApp/Google Docs ne peuvent pas donner, et coûteux à copier crédiblement pour un clone.
- **Émotion** : le bouton merci, les photos du chat — la rétention annuelle passe par le souvenir affectif, pas par les features. (Un email « il y a un an, Miso pendant vos vacances 🧡 » = ta seule campagne de réactivation.)
- Open source : rien de stratégique à ouvrir ici ; éventuellement le format « care sheet » (JSON schema) publié, pour l'effet standard si des sitters pro l'adoptent.

---

## 6. Plan de validation commerciale

### 6.1 Dix cibles précises (canaux plus que comptes nommés — c'est du B2C)

1. **Ta propre garde d'été** (test réel n°1, dixit ta note Notion).
2-3. **2 groupes Facebook FR** « pet sitting entre particuliers » / « gardiennage maison » (posts utiles, pas d'ads).
4. **r/plantedtank & groupes FB plantes d'intérieur FR** (la niche plantes est plus passionnée que la niche chat).
5. **3 créatrices Instagram plantes/chats FR 10-50k** (micro-partenariat : produit gratuit + code).
6. **Nextdoor / groupes de quartier** (garde entre voisins = cœur de cible).
7. **2 conciergeries Airbnb locales** (fiches maison réutilisables — test du segment pro).
8. **1 association de protection animale locale** (familles d'accueil = fiches de garde permanentes, usage intense, feedback riche).
9. **Product Hunt / Bento-like communities** : non prioritaire (audience non FR) — seulement si traduction EN triviale.
10. **SEO immédiat** : 5 pages templates imprimables publiées dès J8 (le trafic « fiche de garde chat PDF » est saisonnier — juillet = maintenant).

### 6.2 Angle & message type (post groupe FB)

> On part 2 semaines en août et j'en avais marre du pavé WhatsApp de 40 lignes pour la voisine qui garde le chat et les plantes. J'ai fait un petit outil : une fiche de garde en ligne, elle ouvre un lien (pas d'app, pas de compte), elle coche matin/soir, je reçois les photos. La première fiche est gratuite : [lien]. Preneur de vos retours — qu'est-ce qui manque dans VOS consignes de garde ?

### 6.3 Offre & prix test

**Gratuit** : 1 fiche complète, 1 garde active, journal inclus. **Pack Veillo 9 € one-shot** (early : 6 €) : fiches illimitées, PDF premium, 2 gardiens+, historique. Segment pro (plus tard) : 12 €/mois conciergeries/sitters multi-foyers — à ne construire QUE si ≥ 10 demandes entrantes.

### 6.4 Validé si (fin septembre — une saison complète)

≥ 300 fiches créées · **taux de viralité mesuré** : ≥ 10 % des gardiens deviennent créateurs de fiche sous 60 j (LA métrique du produit) · ≥ 50 Packs payés (450 €) · ≥ 30 % des gardes avec ≥ 5 coches (le mode garde est réellement utilisé, pas juste la fiche statique).
**Invalidé si** : les fiches sont créées mais les gardiens ne cliquent pas (retour au PDF/WhatsApp) · conversion Pack < 2 % · viralité < 3 %. Dans ce cas : geler en produit gratuit vitrine (il continue de tourner seul, coût ~0) et ne PAS s'acharner — FacturX Kit reste le pari principal.
