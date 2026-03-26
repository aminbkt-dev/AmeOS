# AmeOS — Contexte de reprise pour Claude Code

## Situation actuelle
Projet de data warehouse e-commerce pour la marque AmeOS (Amin Bakhti). Stack : Shopify + Revolut + Supabase (PostgreSQL) + n8n (sync) + Power BI (dashboard).

Le projet est **très avancé** : toutes les données sont en place dans Supabase, les vues SQL sont créées, et Power BI est en cours de construction sur un nouveau PC Windows.

---

## Tables principales dans Supabase

- `drops` — les collections (drop_id, drop_key, drop_name, started_at)
- `drop_variants` — lien variantes Shopify → drops (shopify_variant_id, drop_id, valid_from, valid_to, cost, inventory_item_id)
- `orders` — commandes Shopify (shopify_order_id, ordered_at, total_gross, financial_status, currency, shipping_country_code, shipping_country, shipping_city, shipping_zip)
- `order_items` — lignes de commande (shopify_order_id, shopify_product_id, shopify_variant_id, price, quantity, title, name)
- `returns` — retours (return_id, shopify_order_id, drop_id, refund_type, total_return_amount, reason). `refund_type` = 'PRODUCT_RETURN' ou 'FINANCIAL_REFUND'
- `return_line_items` — lignes de retour (return_id, shopify_variant_id, quantity, refund_amount)
- `expenses` — dépenses Revolut (expense_id, revolut_id, completed_at, type, description, total_amount, category, allocation_status, direction, default_drop_id, mcc, source)
- `expense_allocations` — allocations des dépenses splittées (expense_id, drop_id, category_key, allocated_amount)
- `manual_expenses` + `manual_expense_allocations` — dépenses manuelles
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

---

## Catégories de dépenses

**DROP_COST** (par drop) : PRODUCTION, SHIPPING_PRODUCTION, PACKAGING, SAMPLES, SAMPLES_SHIPPING, CUSTOMS_SAMPLES, SHIPPING_LABELS_FR, SHIPPING_LABELS_INT, SHIPPING_LABELS_RETURNS, PHOTO_SHOOT, VIDEO_MARKETING, TAX_VAT

**MARKETING** (par drop) : ADS_META, ADS_SNAPCHAT, ADS_TIKTOK, INFLUENCER, MARKETING_ACQUISITION

**OVERHEAD** (global, pas par drop) : SAAS, TAXES, STORAGE_BOX, EQUIPMENT, WEBSITE_REDESIGN

**Exclure du P&L** : CASH_FLOW (mouvements de trésorerie), PERSONAL_*, TO_REVIEW

---

## Vues SQL créées dans Supabase

### v_drop_pnl
Vue principale P&L par drop. Une ligne par drop avec :
- ca_brut, nb_orders, aov
- total_returns, nb_returns, return_rate_pct
- ca_net
- transaction_fees (Shopify Payments réels + PayPal 3.62%)
- production, samples, shipping_labels, photo_shoot, video_marketing, tax_vat, ads, influencer
- profit_net, margin_pct

**Note** : profit_net N'INCLUT PAS l'overhead. L'overhead est dans v_overhead séparément.

### v_overhead
Dépenses overhead par mois et catégorie (SAAS, TAXES, STORAGE_BOX, EQUIPMENT, WEBSITE_REDESIGN). Une ligne par dépense avec : expense_id, completed_at, month, category_key, category_label, amount.

### v_product_performance
Performance par variante par drop. Colonnes : drop_key, drop_id, shopify_product_id, shopify_variant_id, product_title, variant_name, sku, units_sold, revenue, cogs (cost × quantity), profit_brut, margin_pct, units_returned, return_rate_pct, refund_amount, stock_available.

### v_orders_detail
Détail par commande pour analyses géo et paiement. Colonnes : shopify_order_id, ordered_at, month, financial_status, total_gross, currency, shipping_country_code, shipping_country, shipping_city, shipping_zip, payment_method, transaction_fee, drop_key.

---

## Connexion Power BI

**Méthode recommandée** : Connexion PostgreSQL directe (sur vrai Windows, pas de problèmes SSL contrairement à une VM (comme ce que j'avais commencé sur mon macbook))

Dans Power BI → Obtenir des données → PostgreSQL :
- **Serveur** : `aws-1-eu-west-1.pooler.supabase.com:5432` (session pooler, compatible IPv4)
- **Base de données** : `postgres`
- **Username** : celui du session pooler (Supabase → Settings → Database → Session pooler)
- **Mot de passe** : mot de passe Supabase
- **Mode** : Import

Importer ces 4 vues : `v_drop_pnl`, `v_overhead`, `v_product_performance`, `v_orders_detail`

**Méthode alternative** : API REST Supabase (si problèmes SSL sur Windows VM)

Dans Power BI → Obtenir des données → Web → Avancé :

**URL** (une par vue) :
```
https://[project_ref].supabase.co/rest/v1/v_drop_pnl?select=*
https://[project_ref].supabase.co/rest/v1/v_overhead?select=*
https://[project_ref].supabase.co/rest/v1/v_product_performance?select=*
https://[project_ref].supabase.co/rest/v1/v_orders_detail?select=*
```

**En-têtes HTTP** (mêmes pour toutes les vues) :
- `apikey` → clé anon public (Supabase → Settings → API)
- `Authorization` → `Bearer [clé_anon_public]`

**Mode** : Import (pas DirectQuery)

---

## État du dashboard Power BI

Page en cours : **Executive Overview**

### KPIs créés (mesures DAX) :
```dax
AOV = SUM(v_drop_pnl[ca_brut]) / SUM(v_drop_pnl[nb_orders])

Marge Nette = FORMAT(
    DIVIDE(SUM(v_drop_pnl[profit_net]), SUM(v_drop_pnl[ca_brut])) * 100,
    "0.0"
) & " %"

Taux Retour = FORMAT(AVERAGE(v_drop_pnl[return_rate_pct]), "0.0") & " %"
```

### Structure du dashboard (4 pages) :
1. **Executive Overview** — KPIs globaux + carte géo + mix paiement + évolution CA
2. **Drops** — P&L complet par drop, comparaison
3. **Produits** — CA/COGS/marge par produit et taille, stock restant
4. **Coûts** — Structure des coûts, overhead, waterfall CA→Profit

### Layout page Executive Overview :
```
[CA Brut] [CA Net] [Profit Net] [Marge %] [Commandes] [AOV] [Taux Retour]
[CA par drop - barres]        [Carte géo - commandes par pays]
[Mix paiement - camembert]    [Évolution CA par mois - courbe]
```

---

## Points importants à savoir

1. **Pas de SKU** dans order_items — les colonnes sku contiennent la chaîne "null" (bug n8n), pas critique pour le dashboard
2. **drop_variants.cost** est en EUR (converti depuis USD avec le taux de change par drop)
3. Les **TikTok orders** apparaissent avec payment_method='unknown' (pas dans tender_transactions)
4. La **Marge %** dans v_drop_pnl est par drop — pour la marge globale utiliser la mesure DAX ci-dessus
5. **profit_net négatif** pour kimono_2026_01 et safar_2025_06 est normal (drop récent + stock non encore écoulé pour kimono)
