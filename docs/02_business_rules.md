# Règles métier — AmeOS / Flowpulse

## Drops (revenus)
- Attribution des ventes Shopify via drop_variants(shopify_variant_id).
- drop_variants gère les overlaps via valid_from / valid_to.
- Un variant peut exister dans plusieurs drops, mais pas sur la même période (fenêtres).

### Exception confirmée
- Pantacourt noir S : variants 51823309324620 et 53397729313100 doivent rester rattachés à cacheawra_2025_01 “à vie” (valid_to = NULL), et ne doivent pas appartenir à d’autres drops.

## Dépenses (coûts)
- Une dépense mono-catégorie + mono-drop : stockée directement dans la table source (category + default_drop_id).
- Les tables allocations sont utilisées UNIQUEMENT en cas de split (multi-catégories et/ou multi-drops).
- Anti double comptage : si une dépense a des allocations, le P&L doit utiliser les allocations et ignorer le montant parent.

## Conventions allocations
- allocation_type = méthode (ex: MANUAL_SPLIT, SHOPIFY_MONTHLY_SPLIT, PRO_RATA…)
- category_key = catégorie analytique du morceau (ex: PRODUCTION, SAMPLES_SHIPPING, PHOTO_SHOOT, etc.)
- drop_id des allocations est indépendant du default_drop_id parent.
  - Exception : propagation autorisée uniquement pour PRODUCTION/SAMPLES quand le parent est mappé et que les allocations n’ont pas de drop.

## Catégorisation
- expense_category_rules sert à auto-assigner expenses.category et éventuellement allocation_status.
- Les règles ne sont pas forcément appliquées automatiquement (selon workflow / step SQL).