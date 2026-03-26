🧠 AmeOS — Data Warehouse & BI Architecture
🎯 Objectif

Construire un système analytique e-commerce permettant :

Calcul du profit réel par drop

Analyse détaillée par produit / taille / pays

Intégration complète des retours

Distinction des méthodes de paiement

Réconciliation cash vs ventes

Base scalable pour futur produit SaaS (Flowpulse)

Le système doit être :

Fiable financièrement

Sans double comptage

Traçable

Audit-ready

Décisionnel

🏗 Architecture Générale
🔹 Sources de données
Shopify

orders

order_items

products

variants

returns

return_line_items

fulfillments

shipping_lines

tender_transactions

transactions

shopify_payments_payouts (si activé)

inventory_items

inventory_levels

Finance

revolut_transactions_raw

expenses

expense_allocations

manual_expenses

manual_expense_allocations

Business Logic

drops

drop_variants (avec valid_from / valid_to)

drop_products

🧩 Logique Métier Fondamentale
1️⃣ Attribution des revenus

Les ventes sont reliées aux drops via :

orders → order_items → shopify_variant_id → drop_variants

Important :

valid_from / valid_to doivent être respectés

Une vente ne doit appartenir qu’à un seul drop

2️⃣ Retours

Les retours :

returns → return_line_items → order_line_item → drop

Les retours impactent :

CA net

Profit net

Taux de retour par drop

Taux de retour par pays

Le profit ne doit jamais être calculé sans intégrer les retours.

3️⃣ Dépenses

Les dépenses sont catégorisées via :

expense_categories

expense_category_rules

Les splits sont gérés via :

expense_allocations

manual_expense_allocations

⚠️ Règle critique :

Une dépense splittée ne doit pas être comptée deux fois.

Si allocations existent :

On ignore le montant parent

On agrège uniquement les allocations

4️⃣ Paiements

Les méthodes de paiement sont déterminées via :

tender_transactions

Objectif :

Distinguer Carte / Apple Pay / PayPal Wallet

Calculer fees par gateway

Analyser impact sur marge

⚠️ Important :

Le champ payment_gateway dans orders n’est pas suffisant.

5️⃣ Cash & Payouts

Les payouts :

Servent uniquement à la réconciliation

Ne doivent jamais servir au calcul du P&L

Règle :

P&L ≠ Cash

P&L = Orders - Refunds - COGS - Fees - Expenses
Cash = Payouts + mouvements bancaires

💰 Logique P&L Officielle
CA brut

Somme des ventes validées

CA net

CA brut - remboursements

Profit brut

CA net - COGS

Profit net

CA net

Production

Shipping labels

Paiements

Marketing

Shooting

Samples

Overhead

📊 Structure Power BI
Page 1 — Executive Overview

KPIs :

CA brut

CA net

Profit net

Marge %

Commandes

AOV

Taux de retour

Mix paiement

Cash reçu

Écart cash vs CA

Page 2 — Drops

Par drop :

CA brut

CA net

Production

Marketing

Fees

Retours

Profit net

Marge %

ROI

Sell-through rate

Page 3 — Produits

CA par produit

Profit par produit

Marge par produit

Ventes par taille

Taux de retour par taille

Stock restant

Rotation stock

Page 4 — Cash & Structure des coûts

Structure % des coûts

Waterfall CA → Profit

Fees par méthode

Délai payout

⚠️ Points Sensibles

Attention au double comptage allocations.

Toujours respecter valid_from / valid_to.

Intégrer retours avant calcul marge.

Ne jamais utiliser payouts pour profit.

Vérifier cohérence CA global = somme CA par drop.

Vérifier que les retours sont bien rattachés aux drops.

🚀 Objectif Long Terme

Transformer ce modèle en produit SaaS :

Flowpulse

Dashboard multi-merchant

Standardisation des KPIs

Modèle réplicable

📌 Philosophie Projet

Ce warehouse doit permettre :

Décisions production

Optimisation marketing

Optimisation mix paiement

Gestion risque retour

Optimisation pays cible

Pilotage cash

🧠 Claude Instructions

Toujours :

Vérifier absence double comptage

Vérifier cohérence P&L

Respecter la logique drop

Prioriser clarté analytique

Signaler incohérences potentielles