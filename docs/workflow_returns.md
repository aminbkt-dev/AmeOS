# Workflow — Rattachement des retours aux drops

## Pourquoi c'est important

`returns.drop_id` est utilisé par `v_drop_pnl` pour calculer le CA net et le taux de retour par drop.
Un retour avec `drop_id = NULL` est invisible dans le P&L.

---

## À chaque import de nouvelles données

### 1. Vérifier les retours non rattachés

```sql
SELECT return_id, created_at, shopify_order_id, total_return_amount, drop_id
FROM returns
WHERE drop_id IS NULL
ORDER BY created_at DESC;
```

### 2. Rattachement automatique via return_line_items

```sql
UPDATE returns r
SET drop_id = subq.drop_id
FROM (
    SELECT DISTINCT ON (r2.return_id)
        r2.return_id,
        d.drop_id
    FROM returns r2
    JOIN return_line_items rli ON r2.return_id = rli.return_id
    JOIN orders o ON r2.shopify_order_id = o.shopify_order_id
    JOIN drop_variants dv ON rli.shopify_variant_id = dv.shopify_variant_id
        AND o.ordered_at BETWEEN dv.valid_from AND COALESCE(dv.valid_to, NOW())
    JOIN drops d ON dv.drop_id = d.drop_id
    WHERE r2.drop_id IS NULL
    ORDER BY r2.return_id
) subq
WHERE r.return_id = subq.return_id;
```

### 3. Vérifier les retours encore NULL après le rattachement automatique

```sql
SELECT
    r.return_id,
    r.shopify_order_id,
    r.created_at,
    r.total_return_amount,
    rli.shopify_variant_id
FROM returns r
LEFT JOIN return_line_items rli ON r.return_id = rli.return_id
WHERE r.drop_id IS NULL
ORDER BY r.created_at;
```

Si `shopify_variant_id = NULL` dans return_line_items → rattachement manuel nécessaire (voir ci-dessous).

### 4. Rattachement manuel (shopify_variant_id manquant)

Identifier le drop via la commande :

```sql
SELECT
    oi.shopify_variant_id,
    oi.title,
    oi.name,
    d.drop_key,
    d.drop_id
FROM order_items oi
JOIN drop_variants dv ON oi.shopify_variant_id = dv.shopify_variant_id
JOIN drops d ON dv.drop_id = d.drop_id
WHERE oi.shopify_order_id = [SHOPIFY_ORDER_ID];
```

Puis assigner manuellement :

```sql
UPDATE returns
SET drop_id = '[DROP_ID]'
WHERE return_id = [RETURN_ID];
```

---

## Règles métier

- Seuls les `refund_type = 'PRODUCT_RETURN'` impactent le CA net et le taux de retour
- Les `FINANCIAL_REFUND` sont des gestes commerciaux — exclus du P&L
- Les retours avec `total_return_amount = 0` sont normaux (retour accepté sans remboursement financier, ou échange de taille)
- Un retour peut avoir `shopify_variant_id = NULL` dans `return_line_items` si le client a commandé plusieurs tailles pour n'en garder qu'une — rattacher manuellement via la commande

---

## Suivi des étiquettes retour en temps réel

Pour éviter de devoir reconstituer l'attribution des étiquettes à posteriori, tenir un fichier de suivi (Google Sheets / Notion) mis à jour à chaque achat d'étiquette :

| Date | Description Revolut | Montant | Drop | Nb étiquettes | Notes |
|------|---------------------|---------|------|----------------|-------|
| 10/03 | Mondial Relay | 16.40€ | ? | 4 | étiquettes achetées en trop ? |

Ce fichier est la source de vérité au moment de l'import — pas besoin de recroiser avec la table `returns`.

---

## Cas particulier — échange de taille

Le client commande deux tailles, renvoie celle qui ne convient pas.
Shopify crée un retour sans variant_id dans return_line_items.
→ Identifier le drop via `order_items` de la commande et assigner manuellement.
