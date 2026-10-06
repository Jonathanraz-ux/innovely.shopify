# Audit & Corrections — Thème Innovely (Shopify)

Rapport du passage d'audit (7 axes), corrections appliquées **sur le checkout principal, non publiées**.

## Méthode
- Revue du code du thème (framework custom elements maison : `product-form.js`, `component.js`, `events.js`, `cart-drawer.js`).
- Vérifications statiques : JSON valides, gates Liquid équilibrées, recherche globale (`5799`, `$57.99`, `/checkout`, `Fermer le panier`, `fbq`/pixel/gtag).
- `templates/index.json` commence par un commentaire `/* … */` : supporté par Shopify (ignoré par les parsers JSON stricts — **ne pas le supprimer**).

---

## A. Défauts confirmés → corrections

### Axe 1 — Panier / boutons
| Statut | Défaut | Correction |
|---|---|---|
| ✔ | « Add to cart » redirigeait vers `/checkout` à chaque ajout (y compris en cas d'erreur, via le `.catch`) : `sections/custom-featured-product.liquid` (ancien script inline, listener en phase capture `preventDefault` + `stopPropagation` qui court-circuitait `on:submit="/handleSubmit"`). | Script inline supprimé → retour au flux natif `product-form-component` (ajout panier via `/cart/add.js`, panier existant préservé, tiroir auto-ouvert, événement `cartLinesUpdate`). |
| ✔ | Erreurs d'ajout silencieuses : le ref `addToCartTextError` requis par `product-form.js` était absent du formulaire custom. | Span `<span class="product-form-text__error hidden" ref="addToCartTextError">` ajouté (avec `icon-error.svg`), conforme au markup natif (`blocks/buy-buttons.liquid`). |
| ✔ | Libellé « Add to cart » vs comportement « Buy now ». | Comportement aligné sur le libellé : ajout panier + tiroir. Checkout réservé au bouton natif de paiement express legits (`{{ form | payment_button }}` : Shop Pay/PayPal) et aux badges de paiement (`snippets/payment-methods-badges.liquid`) — **conservés**. |
| ℹ️ | Tiroir panier ne s'ouvre pas automatiquement. | Réglage thème `auto_open_cart_drawer` (Réglages → Panier), par défaut `false` → **à activer** côté admin. |
| ✔ | Chaîne FR codée en dur « Fermer le panier » (accessibilité) dans le tiroir. | Remplacée par la clé traduite `actions.close` (`snippets/cart-drawer.liquid`). |
| ℹ️ | `templates/cart.json` stocke `"title": "t:text_defaults.cart"`. | Comportement standard : la valeur étant égale au défaut du schéma, Shopify la résout via les locales (`Cart` / `Panier` — clés vérifiées dans `en.default.schema.json` et `fr.schema.json`). **Pas un bug.** |

### Axe 2 — Traduction
| Statut | Défaut | Correction |
|---|---|---|
| ✔ | Textes du tiroir panier en dur en FR. | `actions.close` traduisible (voir Axe 1). |
| ✔ | Témoignage « Achat vérifié / Verified buyer » (rev9), mention « Verified purchase » impossible à vérifier. | Remplacé par une durée neutre : « Client depuis 4 semaines d'utilisation » (`locales/fr.json:102`) / « Customer for 4 weeks » (`locales/en.default.json:102`, `templates/index.json:324`). |
| ℹ️ | Harmonisation EN/FR des sections custom via clés `innovely.*` + `translate.liquid`. | Terminée côté contenu ; à re-vérifier en boutique après activation des toggles (voir E). |

### Axe 3 — Variantes / swatches
| Statut | Défaut | Correction |
|---|---|---|
| ✔ | Radios invisibles (`opacity:0; pointer-events:none`) : aucun indicateur de focus clavier sur les chips. | Règle `:focus-within` + `:has(input:focus-visible)` → contour 2 px `--color-brand` (`sections/custom-featured-product.liquid`). |
| ✔ | Prix barré par variant faux (voir Axe 4). | Synchronisation prix/compare price/image par variant corrigée dans le JSON embarqué (`variant-data-custom`). |

### Axe 4 — Identité / liens
| Statut | Défaut | Correction |
|---|---|---|
| ✔ | Prix barré fabriqué `$57.99` (4 occurrences) : `layout/theme.liquid` (script produit, injectait un `<s>$57.99</s>`), `sections/custom-featured-product.liquid` (`5799` + `'$57.99'`), `sections/custom-products-list.liquid` (badge -% et `{{ 5799 | money }}`), `snippets/price.liquid` (fallback `5799` sur toutes les fiches natives). | Supprimé / remplacé par les `compare_at_price` réels par variant/produit. Badge « -X% » calculé uniquement si `compare_at_price > price`. |
| ℹ️ | Recherche « Innovely-new » | Aucune occurrence dans le code. Aucun lien/identifiant obsolète trouvé. |

### Axe 5 — Promesses commerciales (masquées par défaut, réversibles)
| Contenu masqué | Réglage (éditeur de thème) | Statut |
|---|---|---|
| Note « 4.8/5 · Verified reviews » | Section Custom Featured Product → `show_rating` (`false`) | ✔ |
| Urgence « Limited stock — order now! » | Section Custom Featured Product → `show_urgency` (`false`) | ✔ |
| Stats « +50k / 4.8 / 92% » | Section Custom Social Proof → `show_stats` (`false`) | ✔ |
| Note « 5.0 » sur cartes produits | Section Custom Products List → `show_rating` (`false`) | ✔ |
| Chips hero « 98% » / « From 14 days » | Section Custom Hero → `show_claims` (`false`) | ✔ |
| Stats problème « 8h / 4kg / % » + badge | Section Custom Problem → `show_stats` (`false`) | ✔ |
| Bannière « MONEY-BACK GUARANTEE / 2-YEAR WARRANTY » | Bloc annonce header → `show_text` (`false` réglé dans `sections/header-group.json`) | ✔ |

> Les valeurs restent éditables dans les réglages de sections (rien n'est supprimé du template). **Les toggles se réactivent en éditeur dès que les justificatifs sont réunis (voir D).**

### Axe 6 — Meta / tracking
| Statut | Constat |
|---|---|
| ⚠️ | Aucun script `fbq` / Pixel / `gtag` dans le code du thème (vérifié — les seules correspondances sont des chaînes de coordonnées SVG « pixel ». Ces mesh sont non liés). |
| ⚠️ | Le Pixel Meta semble configuré côté Shopify Admin (Channel). **Ne PAS ajouter de script au thème** (double comptage). Réinstaller/vérifier le pixel + événements standard dans **Canaux de vente → Meta** et activer le **Consent Mode**.

### Axe 7 — Tests
| Statut | Type |
|---|---|
| ✔ | Statiques : JSON valides (commentaires Shopify gérés), gates `if/endif` équilibrées, plus aucune occurrence de `5799`/`$57.99` dans le code produit, plus de redirection `/checkout` inline, `Fermer le panier` traduit. |
| ⏳ | Boutique : à réaliser en brouillon (pas d'accès depuis le repo) — voir **E**. |

---

## B. Fichiers modifiés (corrections d'audit)
- `sections/custom-featured-product.liquid` — flux panier natif, `addToCartTextError`, prix réels, focus chips, toggles `show_rating`/`show_urgency`
- `layout/theme.liquid` — suppression du script d'injection `$57.99`
- `sections/custom-products-list.liquid` — `compare_at_price` réel, toggle `show_rating`
- `snippets/price.liquid` — suppression de la fallback `5799` (fiches natives)
- `sections/custom-social-proof.liquid` — toggle `show_stats`
- `sections/custom-hero.liquid` — toggle `show_claims`
- `sections/custom-problem.liquid` — toggle `show_stats`
- `blocks/_announcement.liquid` + `sections/header-group.json` — toggle `show_text` (bannière garantie)
- `snippets/cart-drawer.liquid` — libellés de fermeture traduits
- `locales/en.default.json`, `locales/fr.json`, `templates/index.json` — neutralisation de la mention « Verified purchase » (rev9)

> À noter : le checkout contient **déjà des modifications non commitées** (travail en cours : traductions `innovely.*`, carrousel social-proof, tiroir panier, footer, images lifestyle/UGC). Les corrections d'audit s'y superposent. **Rien n'est commité ni publié.**

---

## C. Réglages Shopify / Meta à effectuer à la main
1. Réglages thème → Panier → **auto-open cart drawer** : activer (améliore le flux ajout au panier).
2. Vérifier `compare_at_price` réel sur les produits concernés (si aucun prix barré en admin, plus aucun prix barré ne s'affichera — comportement voulu).
3. Meta : pixel unique via Shopify Admin (pas dans le thème), événements standard + ViewContent/AddToCart/Purchase, Consent Mode.
4. Après publication : revisiter chaque toggle de l'Axe 5 selon les justificatifs disponibles.

## D. Justificatifs client à fournir (avant réactivation)
- Note réelle & nombre d'avis (app d'avis, Trustpilot) → remplace 4.8/5 et 5.0.
- Chiffres vérifiables + datés : +50k utilisateurs, 92 %, 98 %, « dès 14 jours », 8h/4kg.
- Politique d'expédition confirmée (livraison offerte) et délais réels.
- Pages Politique de remboursement (argent remboursé) + garantie 2 ans → bannière.
- Durée réelle de la garantie « confort 30 jours » (r2 hero).

## E. Tests à réaliser (brouillon, avant publication)
1. Ajout panier depuis l'accueil : l'article apparaît dans le tiroir (auto-open activé), le panier existant est préservé, pas de redirection /checkout.
2. Variante non disponible : bouton désactivé + libellé « Sold out », erreur visible si ajout impossible.
3. Fiche produit sans `compare_at_price` : aucun prix barré affiché ; avec : prix barré + badge -X% exacts.
4. FR et EN : libellés du tiroir, témoignages, sections custom (toggles off/on).
5. Checkout : bouton natif Shop Pay/PayPal et badges de paiement fonctionnent.
6. Clic sur badge de paiement → `/checkout` (comportement voulu, express).
7. Clavier : navigation/sélection des chips et focus visible.
8. Pixel : Premiers événements (PageView, ViewContent, AddToCart) dans Meta Events Manager ; pas de doublons.

## F. Procédure de rollback
- Tout est **local et non commité** : `git checkout -- <fichiers>` restaure l'état précédent.
- À tout moment en production : publier une version précédente du thème (Boutique en ligne → Thèmes → Historique des versions).
- Les toggles de l'Axe 5 sont des réglages : réactivation/suppression sans code.