# AmeOS — Contexte de reprise pour Claude Code

## Situation actuelle (mise à jour : avril 2026)

Projet de data warehouse e-commerce pour la marque AmeOS (Amin Bakhti). Stack : Shopify + Revolut + Supabase (PostgreSQL) + n8n (sync) + Power BI (dashboard).

**Le dashboard Power BI est construit et fonctionnel** sur Windows natif (pas de VM). Les 5 pages sont en place. Les vues SQL ont été mises à jour pour inclure les manual_orders.

---

## Tables principales dans Supabase

- `drops` — les collections (drop_id, drop_key, drop_name, started_at)
- `drop_variants` — lien variantes Shopify → drops (shopify_variant_id, drop_id, valid_from, valid_to, cost, inventory_item_id)
- `orders` — commandes Shopify (shopify_order_id, ordered_at, total_gross, financial_status, currency, shipping_country_code, shipping_country, shipping_city, shipping_zip)
- `order_items` — lignes de commande (shopify_order_id, shopify_product_id, shopify_variant_id, price, quantity, title, name)
- `returns` — retours (return_id, shopify_order_id, drop_id, refund_type, total_return_amount, reason). `refund_type` = 'PRODUCT_RETURN' ou 'FINANCIAL_REFUND'
- `return_line_items` — lignes de retour (return_id, shopify_variant_id, quantity, refund_amount)
- `expenses` — dépenses Revolut (expense_id, revolut_id, started_at, completed_at, type, description, total_amount, category, allocation_status, direction, default_drop_id, mcc, source)
- `expense_allocations` — allocations des dépenses splittées (expense_id, drop_id, category_key, allocated_amount, allocation_type)
- `manual_expenses` + `manual_expense_allocations` — dépenses manuelles
- `manual_orders` — commandes manuelles (manual_order_id uuid, ordered_at, drop_id, financial_status, total_gross, currency, notes)
- `expense_categories` — catégories (category_key, category_label, category_group, is_business)
- `tender_transactions` — méthodes de paiement par commande (shopify_order_id, payment_method: credit_card/apple_pay/paypal/unknown)
- `shopify_payment_transactions` — frais de transaction Shopify Payments (shopify_order_id, type, fee). Utiliser type='charge' pour les frais
- `inventory_items` — stock Shopify (inventory_item_id, sku, cost)
- `inventory_levels` — niveaux de stock (inventory_item_id, location_id, available)

---

## Règles métier critiques

### Attribution des revenus
- Commandes → order_items → drop_variants (via shopify_variant_id + valid_from/valid_to)
- Pour les commandes multi-drops : revenus distribués proportionnellement par item_revenue (price × quantity)

### Dépenses — anti double comptage
- Si `allocation_status` IN ('ALLOCATED', 'SPLIT_DONE') → utiliser UNIQUEMENT expense_allocations (ignorer le montant parent)
- Si `allocation_status` = 'UNALLOCATED' → utiliser expenses.total_amount directement

### Frais de transaction
- PayPal : estimation 3.62% du total_gross
- Shopify Payments : frais réels depuis shopify_payment_transactions WHERE type='charge'
- Méthode déterminée via tender_transactions.payment_method

### COGS vs Production
- `drop_variants.cost` = coût unitaire par variante (en EUR, converti depuis USD au taux de change du drop) → utilisé pour la page Produits
- `PRODUCTION` dans expense_categories = virement total à l'usine → utilisé pour le P&L par drop

### Retours
- Seuls les `refund_type = 'PRODUCT_RETURN'` impactent le CA net et le taux de retour
- Les `FINANCIAL_REFUND` sont des gestes commerciaux/remboursements financiers, exclus du CA
- `returns.drop_id` doit toujours être renseigné — rattachement automatique via return_line_items, manuel sinon
- Voir workflow_returns.md pour la procédure complète

---

## Catégories de dépenses

**DROP_COST** (par drop) : PRODUCTION, SHIPPING_PRODUCTION, PACKAGING, SAMPLES, SAMPLES_SHIPPING, CUSTOMS_SAMPLES, SHIPPING_LABELS_FR, SHIPPING_LABELS_INT, SHIPPING_LABELS_RETURNS, PHOTO_SHOOT, VIDEO_MARKETING, TAX_VAT

**MARKETING** (par drop) : ADS_META, ADS_SNAPCHAT, ADS_TIKTOK, INFLUENCER, MARKETING_ACQUISITION

**OVERHEAD** (global, pas par drop) : SAAS, TAXES, STORAGE_BOX, EQUIPMENT, WEBSITE_REDESIGN

**Exclure du P&L** : CASH_FLOW (mouvements de trésorerie), PERSONAL_SALARY, PERSONAL_RESTAURANTS, PERSONAL_TRAVEL, TO_REVIEW, OTHER

---

## Vues SQL créées dans Supabase

### v_drop_pnl
Vue principale P&L par drop. Une ligne par drop avec :
- ca_brut, nb_orders, aov
- total_returns, nb_returns, return_rate_pct
- ca_net
- transaction_fees (Shopify Payments réels + PayPal 3.62%)
- production, packaging, shipping_production (séparés), samples, shipping_labels, photo_shoot, video_marketing, tax_vat, ads, influencer
- profit_net, margin_pct

**Note** : profit_net N'INCLUT PAS l'overhead. L'overhead est dans v_overhead séparément.
**Note** : inclut les manual_orders via CTE combined_revenue (UNION entre drop_revenue et manual_drop_revenue).

### v_overhead
Dépenses overhead par mois et catégorie (SAAS, TAXES, STORAGE_BOX, EQUIPMENT, WEBSITE_REDESIGN). Une ligne par dépense avec : expense_id, completed_at, month, category_key, category_label, amount.

### v_product_performance
Performance par variante par drop. Colonnes : drop_key, drop_id, shopify_product_id, shopify_variant_id, product_title, variant_name, sku, units_sold, revenue, cogs (cost × quantity), profit_brut, margin_pct, units_returned, return_rate_pct, refund_amount, stock_available.

### v_orders_detail
Détail par commande pour analyses géo et paiement. Colonnes : shopify_order_id, ordered_at, month, financial_status, total_gross, currency, shipping_country_code, shipping_country, shipping_city, shipping_zip, payment_method, transaction_fee, drop_key, manual_order_id.
Inclut les manual_orders via UNION (géo = France par défaut, payment_method = 'manual').

---

## Connexion Power BI

**Méthode utilisée** : API REST Supabase (le connecteur PostgreSQL direct bloquait sur une erreur SSL de certificat)

Dans Power BI → Obtenir des données → Web → Avancé :

**URL** (une par vue) :
```
https://[project_ref].supabase.co/rest/v1/v_drop_pnl?select=*
https://[project_ref].supabase.co/rest/v1/v_overhead?select=*
https://[project_ref].supabase.co/rest/v1/v_product_performance?select=*
https://[project_ref].supabase.co/rest/v1/v_orders_detail?select=*
```

**Pour la table expenses (page Perso & Divers)** :
```
https://[project_ref].supabase.co/rest/v1/expenses?select=expense_id,started_at,completed_at,description,total_amount,category,allocation_status,default_drop_id&category=in.(PERSONAL_SALARY,PERSONAL_RESTAURANTS,PERSONAL_TRAVEL,OTHER,TO_REVIEW)
```
→ Filtrer ensuite dans Power Query : colonne `description` → Ne contient pas → `paypal`

**En-têtes HTTP** (mêmes pour toutes les sources) :
- `apikey` → clé anon public (Supabase → Settings → API)
- `Authorization` → `Bearer [clé_anon_public]`

**Mode** : Import (pas DirectQuery)

**Important** : après import, dans Power Query → étape "Column1 développée" → mettre à jour la liste des colonnes si la vue a évolué. Les colonnes numériques (total_amount, packaging, shipping_production, etc.) doivent être typées en **Nombre décimal**.

---

## Mesures DAX créées

```dax
CA Brut = SUM(v_drop_pnl[ca_brut])

CA Net = SUM(v_drop_pnl[ca_net])

Profit Net = SUM(v_drop_pnl[profit_net])

Marge Nette = FORMAT(DIVIDE(SUM(v_drop_pnl[profit_net]), SUM(v_drop_pnl[ca_brut])) * 100, "0.0") & " %"

Nb Commandes = COUNTROWS(v_orders_detail)

AOV = DIVIDE(SUM(v_drop_pnl[ca_brut]), SUM(v_drop_pnl[nb_orders]))

Taux Retour = FORMAT(AVERAGE(v_drop_pnl[return_rate_pct]), "0.0") & " %"

Nb Commandes Geo = DISTINCTCOUNT(v_orders_detail[shopify_order_id])

CA Brut Mensuel = SUM(v_orders_detail[total_gross])

Commandes Par Pays = COUNTROWS(v_orders_detail)

Taux Retour Produit = DIVIDE(SUM(v_product_performance[units_returned]), SUM(v_product_performance[units_sold])) * 100

Total Perso & Divers = SUM(expenses[total_amount])

Valeur Waterfall = SWITCH(
    SELECTEDVALUE(Waterfall[Étape]),
    "CA Brut", SUM(v_drop_pnl[ca_brut]),
    "Retours", -SUM(v_drop_pnl[total_returns]),
    "Fees", -SUM(v_drop_pnl[transaction_fees]),
    "Production", -SUM(v_drop_pnl[production]),
    "Packaging", -SUM(v_drop_pnl[packaging]),
    "Shipping Production", -SUM(v_drop_pnl[shipping_production]),
    "Samples", -SUM(v_drop_pnl[samples]),
    "Shipping Labels", -SUM(v_drop_pnl[shipping_labels]),
    "Ads", -SUM(v_drop_pnl[ads]),
    "Influencer", -SUM(v_drop_pnl[influencer]),
    "Photo Shoot", -SUM(v_drop_pnl[photo_shoot]),
    "Vidéo", -SUM(v_drop_pnl[video_marketing]),
    "Tax VAT", -SUM(v_drop_pnl[tax_vat])
)
```

### Table calculée Waterfall
```dax
Waterfall = DATATABLE(
    "Étape", STRING,
    "Ordre", INTEGER,
    {
        {"CA Brut", 1},
        {"Retours", 2},
        {"Fees", 3},
        {"Production", 4},
        {"Packaging", 5},
        {"Shipping Production", 6},
        {"Samples", 7},
        {"Shipping Labels", 8},
        {"Ads", 9},
        {"Influencer", 10},
        {"Photo Shoot", 11},
        {"Vidéo", 12},
        {"Tax VAT", 13}
    }
)
```

---

## Structure du dashboard Power BI (5 pages)

### Page 1 — Executive Overview
- 7 cartes KPI : CA Brut, CA Net, Profit Net, Marge Nette, Nb Commandes, AOV, Taux Retour
- Histogramme CA Brut par drop (v_drop_pnl → drop_key / CA Brut)
- Graphique à barres horizontal commandes par pays (v_orders_detail → shipping_country / Commandes Par Pays)
- Camembert mix paiement (v_orders_detail → payment_method / Nb Commandes Geo)
- Courbe évolution CA mensuel (v_orders_detail → month / CA Brut Mensuel)

### Page 2 — Drops
- Tableau P&L complet par drop (toutes colonnes v_drop_pnl)
- Histogramme groupé profit_net par drop (couleurs conditionnelles positif/négatif + ligne constante à 0)
- Anneau répartition CA Brut par drop
- Histogramme empilé structure des coûts par drop (production, packaging, shipping_production, samples, shipping_labels, ads, influencer, photo_shoot, video_marketing, tax_vat)

### Page 3 — Produits
- Segment filtre par drop_key
- Tableau détail par variante (v_product_performance)
- Histogramme groupé revenue + profit_brut par product_title
- Histogramme units_sold par variant_name (ventes par taille)
- Histogramme Taux Retour Produit par product_title

### Page 4 — Coûts
- Histogramme empilé overhead par mois (v_overhead → month / amount / category_key)
- Anneau overhead par catégorie (v_overhead → category_key / amount) + carte KPI total superposée
- Tableau détail overhead (v_overhead → month, category_label, amount)
- Graphique en cascade CA → Profit (table Waterfall + mesure Valeur Waterfall)

### Page 5 — Perso & Divers
- Carte KPI Total Perso & Divers
- Segment filtre par category
- Tableau dépenses (expenses → started_at, description, total_amount, category)
- Histogramme Total Perso & Divers par category

---

## Points importants à savoir

1. **Pas de SKU** dans order_items — les colonnes sku contiennent la chaîne "null" (bug n8n), pas critique
2. **drop_variants.cost** est en EUR (converti depuis USD avec le taux de change par drop)
3. Les **TikTok orders** apparaissent avec payment_method='unknown' (pas dans tender_transactions)
4. La **Marge %** dans v_drop_pnl est par drop — pour la marge globale utiliser la mesure DAX
5. **profit_net négatif** pour kimono_2026_01 est normal (drop récent, stock non encore écoulé)
6. `returns` ne contient que les retours clôturés (status = 'refunded') — les demandes en cours n'apparaissent pas
7. `started_at` dans expenses = date réelle du paiement (à utiliser pour les fenêtres d'allocation). `completed_at` peut être en retard de 1-2 jours ouvrés
8. Les dépenses Mondial Relay (SHIPPING_LABELS_RETURNS) doivent être notées manuellement à chaque achat (drop associé) — voir workflow_returns.md
9. Voir workflow_shipping_labels.md pour la procédure d'allocation des étiquettes d'expédition

---

## Workflows documentés

- `docs/workflow_shipping_labels.md` — allocation des étiquettes d'expédition FR/INT par drop (fenêtres, pro-rata commandes)
- `docs/workflow_returns.md` — rattachement des retours aux drops + suivi des étiquettes retour
