# Schema BDD - app_db

Base de donnees applicative pour la verification des declarations DJI.

**Connexion :** `localhost:5433` | User: `app_user` | DB: `app_db`

---

## Diagramme relationnel

```
┌─────────────────────────────┐
│    prestashop_customers     │
│  PK: id_customer            │
│  siret, email, company      │
└──────────────┬──────────────┘
               │
       ┌───────┴───────┐
       │               │
       ▼               ▼
┌─────────────┐   ┌───────────────────────┐
│ declarations│   │  factures_revendeurs  │
│FK:id_customer│   │  FK: id_customer      │
│ reference   │   │  vendeur_siret        │
│ status      │   └───────────┬───────────┘
└─────────────┘               │
                              ▼
                 ┌────────────────────────────┐
                 │ factures_revendeurs_lignes │
                 │ FK: facture_id             │
                 │ FK: promo_reference_article│
                 └────────────┬───────────────┘
                              │
                              ▼
                 ┌────────────────────────────┐
                 │      promotions_dji        │
                 │  reference_article (UK)    │
                 │  prix, compensation        │
                 └────────────────────────────┘
```

---

## Tables

### 1. prestashop_customers

Table pivot centrale - clients synchronises depuis l'API PrestaShop.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id_customer | INTEGER | PK | ID PrestaShop du client |
| firstname | VARCHAR(100) | NOT NULL | Prenom |
| lastname | VARCHAR(100) | NOT NULL | Nom |
| email | VARCHAR(255) | NOT NULL, UNIQUE | Email client |
| company | VARCHAR(200) | | Raison sociale |
| siret | VARCHAR(14) | | SIRET - pour matcher les factures |
| customer_since | TIMESTAMPTZ | | Date inscription PrestaShop |
| s3_path | VARCHAR(255) | | Chemin S3 racine du client |
| declarations_count | INTEGER | DEFAULT 0 | Nombre de declarations |
| last_sync_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Derniere sync API |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**Index :** `siret`, `email`, `company`

---

### 2. declarations

Cache local des declarations PrestaShop.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id_declaration | INTEGER | PK | ID declaration PrestaShop |
| id_customer | INTEGER | FK → prestashop_customers, NOT NULL | Client proprietaire |
| reference | VARCHAR(50) | NOT NULL, UNIQUE | Reference (ex: DJI-A1B2C3D4) |
| status | VARCHAR(30) | NOT NULL | draft, submitted, processing, validated, rejected |
| status_label | VARCHAR(50) | | Libelle affichable |
| has_mic_power_products | BOOLEAN | DEFAULT FALSE | Necessite photos stock |
| xlsx_filename | VARCHAR(255) | | Nom du fichier XLSX |
| s3_path | VARCHAR(255) | | Chemin S3 des fichiers |
| date_add | TIMESTAMPTZ | | Date creation PrestaShop |
| date_upd | TIMESTAMPTZ | | Date MAJ PrestaShop |
| last_sync_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**Index :** `id_customer`, `status`, `reference`

---

### 3. promotions_dji

Promotions et baisses de prix DJI (importees depuis Excel).

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| statut | VARCHAR(50) | | Statut de la promo |
| type_promotion | VARCHAR(100) | | Type (baisse prix, promo...) |
| produit | TEXT | NOT NULL | Designation produit |
| ean | VARCHAR(30) | | Code EAN |
| reference_article | VARCHAR(50) | | Reference DJI (cle de matching) |
| date_debut | DATE | | Debut de validite |
| date_fin | DATE | | Fin de validite |
| prix_vente_ttc | NUMERIC(10,2) | | Ancien prix TTC |
| nouveau_prix_ttc | NUMERIC(10,2) | | Nouveau prix TTC |
| difference | NUMERIC(10,2) | | Ecart de prix |
| compensation_ht | NUMERIC(10,2) | | Compensation HT versee par DJI |
| date_butoir_declaration | DATE | | Date limite pour declarer |
| mise_a_jour | DATE | | Date MAJ du fichier source |
| declaration_sts_dji | VARCHAR(100) | | Statut declaration STS |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**Index :** `reference_article`, `statut`, `type_promotion`, `(date_debut, date_fin)`

---

### 4. factures_revendeurs

En-tetes des factures PDF extraites via Mistral.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id | SERIAL | PK | |
| id_customer | INTEGER | FK → prestashop_customers | Lien client (optionnel) |
| facture_numero | VARCHAR(100) | NOT NULL | Numero de facture |
| facture_date | DATE | NOT NULL | Date de facturation |
| vendeur_nom | VARCHAR(200) | NOT NULL | Nom du vendeur |
| vendeur_siret | VARCHAR(14) | | SIRET vendeur |
| vendeur_tva | VARCHAR(20) | | N° TVA intracommunautaire |
| client_nom | VARCHAR(200) | | Nom client sur facture |
| client_adresse | TEXT | | Adresse client |
| total_ht | NUMERIC(10,2) | | Total HT |
| total_tva | NUMERIC(10,2) | | Total TVA |
| total_ttc | NUMERIC(10,2) | | Total TTC |
| reste_a_payer | NUMERIC(10,2) | | Reste a payer |
| mode_paiement | VARCHAR(50) | | Mode de paiement |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**Contrainte unique :** `(facture_numero, vendeur_siret)`

**Index :** `facture_numero`, `vendeur_siret`, `facture_date`, `id_customer`

---

### 5. factures_revendeurs_lignes

Lignes articles des factures (relation 1-N avec factures_revendeurs).

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id | SERIAL | PK | |
| facture_id | INTEGER | FK → factures_revendeurs, NOT NULL, ON DELETE CASCADE | |
| designation | TEXT | NOT NULL | Libelle produit |
| ean | VARCHAR(30) | | Code EAN |
| numero_serie | VARCHAR(30) | | Numero de serie |
| code_article | VARCHAR(50) | | Code article |
| quantite | NUMERIC(10,3) | NOT NULL | Quantite (negatif = avoir) |
| prix_ht | NUMERIC(10,2) | | Prix unitaire HT |
| prix_ttc | NUMERIC(10,2) | | Prix unitaire TTC |
| promo_reference_article | VARCHAR(50) | FK → promotions_dji.reference_article | Reference DJI matchee |
| controle_statut | VARCHAR(30) | | OK, ECART_PRIX, PROMO_INCONNUE, HORS_PERIODE, AVOIR |
| controle_ecart_ttc | NUMERIC(10,2) | | Ecart prix facture vs promo |
| controle_date | TIMESTAMPTZ | | Date du controle |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**Index :** `facture_id`, `promo_reference_article`, `controle_statut`

---

## Workflow de verification

```
1. SYNC prestashop_customers   ← API PrestaShop /customers
         ↓
2. SYNC declarations           ← API PrestaShop /customer?id=X
         ↓
3. IMPORT promotions_dji       ← Fichier Excel DJI
         ↓
4. EXTRACT factures PDF        ← Mistral AI → factures_revendeurs + lignes
         ↓
5. MATCHING lignes ↔ promos    ← JOIN sur EAN ou reference_article
         ↓
6. CONTROLE prix               ← Compare prix_ttc vs nouveau_prix_ttc
         ↓
7. RAPPORT anomalies           ← WHERE controle_statut != 'OK'
```

---

## Requetes utiles

**Anomalies de prix par client :**
```sql
SELECT 
    c.company,
    c.email,
    f.facture_numero,
    l.designation,
    l.prix_ttc AS prix_facture,
    p.nouveau_prix_ttc AS prix_promo,
    l.controle_ecart_ttc
FROM factures_revendeurs_lignes l
JOIN factures_revendeurs f ON f.id = l.facture_id
JOIN prestashop_customers c ON c.id_customer = f.id_customer
LEFT JOIN promotions_dji p ON p.reference_article = l.promo_reference_article
WHERE l.controle_statut = 'ECART_PRIX'
ORDER BY c.company, f.facture_date;
```

**Declarations non verifiees :**
```sql
SELECT 
    c.company,
    d.reference,
    d.status,
    d.date_add
FROM declarations d
JOIN prestashop_customers c ON c.id_customer = d.id_customer
WHERE d.status = 'submitted'
  AND NOT EXISTS (
      SELECT 1 FROM factures_revendeurs f 
      WHERE f.id_customer = d.id_customer
  );
```
