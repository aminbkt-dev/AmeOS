# AmeOS / Flowpulse — Warehouse V1

## Objectif
Calculer CA / marge / profit par drop en combinant :
- Shopify : orders, order_items, products, variants, inventory
- Revolut : expenses (normalisées) + staging brut
- Données manuelles : manual_orders / manual_expenses

## Principes clés
- Revenus Shopify -> attribution drop via drop_variants (valid_from/valid_to).
- Dépenses mono-drop -> default_drop_id directement sur la dépense.
- Dépenses splittées -> allocations uniquement (expense_allocations / manual_expense_allocations).
- Jamais de double comptage : allocations remplacent le montant parent dans les KPI.

## Fichiers importants
- database/schema.sql : export du schéma Supabase
- docs/00_context_init.md : prompt et contraintes pour l’assistant
- docs/02_business_rules.md : règles métier et exceptions
- queries/ : requêtes QA & KPI