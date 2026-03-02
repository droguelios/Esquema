# Esquema
megastore-api/
│
├── src/
│   ├── config/
│   ├── controllers/
│   ├── routes/
│   ├── services/
│   ├── migrations/
│   └── utils/
│
├── docs/
│   ├── ERD.png
│   ├── collections.png
│   ├── sample_data.csv
│   ├── sql_schema.sql
│   └── mongo_validation.js
│
├── .env
├── README.md
└── package.json
src/
 ├── config/
 │    ├── mysql.js
 │    └── mongo.js
 ├── controllers/
 │    └── productController.js
 ├── routes/
 │    └── productRoutes.js
 ├── services/
 │    └── migrationService.js
 └── app.js

 mysql.jsCustomers
---------
customer_id (PK)
name
email (UNIQUE)
address

Suppliers
---------
supplier_id (PK)
name
contact

Categories
----------
category_id (PK)
name (UNIQUE)

Products
--------
product_id (PK)
sku (UNIQUE)
name
price
category_id (FK)
supplier_id (FK)

Orders
------
order_id (PK)
transaction_id (UNIQUE)
date
customer_id (FK)

OrderItems
----------
order_item_id (PK)
order_id (FK)
product_id (FK)
quantity
unit_price


megastore-api/
│
├── src/
│   ├── config/
│   │    ├── mysql.js
│   │    └── mongo.js
│   │
│   ├── controllers/
│   │    ├── productController.js
│   │    └── analyticsController.js
│   │
│   ├── routes/
│   │    ├── productRoutes.js
│   │    └── analyticsRoutes.js
│   │
│   ├── services/
│   │    └── migrationService.js
│   │
│   └── app.js
│
├── docs/
│   ├── sql_schema.sql
│   ├── mongo_validation.js
│   └── sample_data.csv
│
├── .env
├── package.json
└── README.md

# MegaStore Global Migration Project

## Architecture

### SQL (MySQL)
Used for transactional and relational data.
Normalized to 3NF to eliminate redundancy and ensure integrity.

### MongoDB
Used for audit logs.
Deletion snapshots are embedded to preserve historical traceability.

## Migration

POST /migrate

The migration process is idempotent.
Entities are checked before insertion using UNIQUE constraints and lookups.

## CRUD

Entity implemented: Products

Endpoints:
GET /products
POST /products
PUT /products/:id
DELETE /products/:id

## Business Intelligence

GET /analytics/suppliers
GET /analytics/customer/:email
GET /analytics/top-products/:category
