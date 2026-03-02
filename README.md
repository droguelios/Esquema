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


const pool = require('../config/mysql');

// 1️⃣ Proveedores con más ventas
exports.topSuppliers = async (req, res) => {
  const [rows] = await pool.query(`
    SELECT s.name,
           SUM(oi.quantity) AS total_items,
           SUM(oi.quantity * oi.unit_price) AS total_value
    FROM suppliers s
    JOIN products p ON s.supplier_id = p.supplier_id
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY s.name
    ORDER BY total_items DESC
  `);
  res.json(rows);
};

// 2️⃣ Historial de cliente
exports.customerHistory = async (req, res) => {
  const [rows] = await pool.query(`
    SELECT o.transaction_id,
           o.date,
           p.name,
           oi.quantity,
           (oi.quantity * oi.unit_price) AS total
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE c.email = ?
  `, [req.params.email]);

  res.json(rows);
};

// 3️⃣ Productos estrella por categoría
exports.topProductsByCategory = async (req, res) => {
  const [rows] = await pool.query(`
    SELECT p.name,
           SUM(oi.quantity * oi.unit_price) AS revenue
    FROM products p
    JOIN categories c ON p.category_id = c.category_id
    JOIN order_items oi ON p.product_id = oi.product_id
    WHERE c.name = ?
    GROUP BY p.name
    ORDER BY revenue DESC
  `, [req.params.category]);

  res.json(rows);
};

const fs = require('fs');
const csv = require('csv-parser');
const pool = require('../config/mysql');

async function migrateCSV(filePath) {

  return new Promise((resolve, reject) => {

    fs.createReadStream(filePath)
      .pipe(csv())
      .on('data', async (row) => {

        try {

          // 1️⃣ CUSTOMER
          const [customer] = await pool.query(
            "SELECT customer_id FROM customers WHERE email=?",
            [row["Email Cliente"]]
          );

          let customerId;

          if (customer.length === 0) {
            const [result] = await pool.query(
              "INSERT INTO customers (name,email,address) VALUES (?,?,?)",
              [
                row["Nombre Cliente"],
                row["Email Cliente"],
                row["Dirección"]
              ]
            );
            customerId = result.insertId;
          } else {
            customerId = customer[0].customer_id;
          }

          // 2️⃣ SUPPLIER
          const [supplier] = await pool.query(
            "SELECT supplier_id FROM suppliers WHERE name=?",
            [row["Nombre Proveedor"]]
          );

          let supplierId;

          if (supplier.length === 0) {
            const [result] = await pool.query(
              "INSERT INTO suppliers (name,contact) VALUES (?,?)",
              [
                row["Nombre Proveedor"],
                row["Contacto Proveedor"]
              ]
            );
            supplierId = result.insertId;
          } else {
            supplierId = supplier[0].supplier_id;
          }

          // 3️⃣ CATEGORY
          const [category] = await pool.query(
            "SELECT category_id FROM categories WHERE name=?",
            [row["Categoría Producto"]]
          );

          let categoryId;

          if (category.length === 0) {
            const [result] = await pool.query(
              "INSERT INTO categories (name) VALUES (?)",
              [row["Categoría Producto"]]
            );
            categoryId = result.insertId;
          } else {
            categoryId = category[0].category_id;
          }

          // 4️⃣ PRODUCT
          const [product] = await pool.query(
            "SELECT product_id FROM products WHERE sku=?",
            [row["SKU"]]
          );

          let productId;

          if (product.length === 0) {
            const [result] = await pool.query(
              `INSERT INTO products (sku,name,price,category_id,supplier_id)
               VALUES (?,?,?,?,?)`,
              [
                row["SKU"],
                row["Nombre Producto"],
                row["Precio Unitario"],
                categoryId,
                supplierId
              ]
            );
            productId = result.insertId;
          } else {
            productId = product[0].product_id;
          }

          // 5️⃣ ORDER
          const [order] = await pool.query(
            "SELECT order_id FROM orders WHERE transaction_id=?",
            [row["ID Transacción"]]
          );

          let orderId;

          if (order.length === 0) {
            const [result] = await pool.query(
              `INSERT INTO orders (transaction_id,date,customer_id)
               VALUES (?,?,?)`,
              [
                row["ID Transacción"],
                row["Fecha"],
                customerId
              ]
            );
            orderId = result.insertId;
          } else {
            orderId = order[0].order_id;
          }

          // 6️⃣ ORDER ITEMS
          await pool.query(
            `INSERT INTO order_items (order_id,product_id,quantity,unit_price)
             VALUES (?,?,?,?)`,
            [
              orderId,
              productId,
              row["Cantidad"],
              row["Precio Unitario"]
            ]
          );

        } catch (err) {
          console.error(err);
        }

      })
      .on('end', resolve)
      .on('error', reject);
  });
}

module.exports = migrateCSV;
