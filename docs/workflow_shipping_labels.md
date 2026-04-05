# Workflow — Allocation des étiquettes d'expédition (SHIPPING_LABELS_FR / INT)

## Principe de la fenêtre

Chaque dépense d'étiquettes couvre les commandes passées entre deux paiements.
La fenêtre d'une dépense = **[lendemain du started_at du batch précédent, veille du started_at du batch actuel]**.

On utilise `started_at` (et non `completed_at`) car Revolut peut mettre 1-2 jours ouvrés à compléter une transaction. `started_at` = moment réel du paiement.

On exclut les jours aux deux bornes car `started_at` est stocké à minuit : on ne sait pas si les commandes du même jour ont eu lieu avant ou après le paiement.

---

## Étapes à chaque import de dépenses

### 1. Identifier les nouvelles dépenses d'étiquettes

```sql
SELECT expense_id, started_at, completed_at, description, total_amount, allocation_status
FROM expenses
WHERE category IN ('SHIPPING_LABELS_FR', 'SHIPPING_LABELS_INT')
  AND allocation_status = 'UNALLOCATED'
ORDER BY started_at;
```

### 2. Identifier la dernière dépense déjà allouée (borne de début)

```sql
SELECT expense_id, started_at, description, total_amount
FROM expenses
WHERE category IN ('SHIPPING_LABELS_FR', 'SHIPPING_LABELS_INT')
  AND allocation_status IN ('ALLOCATED', 'SPLIT_DONE')
ORDER BY started_at DESC
LIMIT 1;
```

→ Noter son `expense_id` et son `started_at`. C'est la **borne de début**.

### 3. Calculer le pro-rata commandes par drop sur la fenêtre

```sql
SELECT
    d.drop_key,
    d.drop_id,
    COUNT(DISTINCT o.shopify_order_id) AS nb_orders
FROM orders o
JOIN order_items oi ON o.shopify_order_id = oi.shopify_order_id
JOIN drop_variants dv ON oi.shopify_variant_id = dv.shopify_variant_id
    AND o.ordered_at BETWEEN dv.valid_from AND COALESCE(dv.valid_to, NOW())
JOIN drops d ON dv.drop_id = d.drop_id
WHERE o.ordered_at >= (
    SELECT DATE(started_at) + 1
    FROM expenses
    WHERE expense_id = '[EXPENSE_ID_DERNIERE_ALLOUEE]'
)
AND o.ordered_at < (
    SELECT DATE(started_at)
    FROM expenses
    WHERE expense_id = '[EXPENSE_ID_DERNIERE_DU_NOUVEAU_BATCH]'
)
AND o.financial_status != 'refunded'
GROUP BY d.drop_key, d.drop_id
ORDER BY nb_orders DESC;
```

**Borne de fin** = `started_at` de la **dernière** dépense du nouveau batch (pas d'aujourd'hui — les commandes récentes dont les étiquettes ne sont pas encore achetées ne doivent pas être incluses).

### 4. Calculer les parts (%)

```
total_commandes = somme des nb_orders
part_drop_X = nb_orders_drop_X / total_commandes
```

### 5. Cas particulier — dépense mixte (ex: abonnement Shopify + étiquettes)

Si une dépense contient plusieurs catégories (ex: SAAS + SHIPPING_LABELS_FR) :
- Identifier le montant fixe (ex: 36€ abonnement)
- Soustraire du total → montant net à splitter en étiquettes
- Créer une allocation SAAS sans drop_id (overhead)
- Créer les allocations SHIPPING_LABELS_FR avec le montant net

### 6. Insérer les allocations

```sql
WITH expense_data AS (
  SELECT unnest(ARRAY['...']::uuid[]) AS expense_id,
         unnest(ARRAY[...]) AS total_amount
),
drop_shares AS (
  SELECT unnest(ARRAY['...']::uuid[]) AS drop_id,
         unnest(ARRAY[...]) AS share
)
INSERT INTO expense_allocations (expense_id, drop_id, category_key, allocated_amount)
SELECT e.expense_id, d.drop_id, 'SHIPPING_LABELS_FR',
       ROUND((e.total_amount * d.share)::numeric, 2)
FROM expense_data e
CROSS JOIN drop_shares d;
```

### 7. Mettre à jour allocation_status

```sql
UPDATE expenses
SET allocation_status = 'SPLIT_DONE'
WHERE expense_id IN ('...', '...');
```

### 8. Vérification

```sql
SELECT
    e.description,
    e.total_amount,
    d.drop_key,
    ea.category_key,
    ea.allocated_amount
FROM expense_allocations ea
JOIN expenses e ON ea.expense_id = e.expense_id
JOIN drops d ON ea.drop_id = d.drop_id
WHERE ea.expense_id IN ('...', '...')
ORDER BY e.description, d.drop_key;
```

---

## Règles à ne pas oublier

- Toujours utiliser `started_at`, jamais `completed_at`
- Exclure les jours aux deux bornes (`DATE + 1` et `< DATE`, pas `<=`)
- La borne de fin = `started_at` de la dernière dépense du batch actuel (pas aujourd'hui)
- Les commandes récentes sans étiquettes encore achetées restent pour le prochain batch
- `allocation_status = 'SPLIT_DONE'` obligatoire sinon la vue v_drop_pnl double-compte
- SHIPPING_LABELS_INT suit le même workflow mais avec les commandes internationales uniquement
