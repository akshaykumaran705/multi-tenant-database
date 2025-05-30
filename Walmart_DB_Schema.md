# PostgreSQL Retail Database Project: Walmart Case Study

This document outlines the schema, procedures, triggers, Row-Level Security, batch ETL, and real-time streaming setup used to support a multi-tenant retail transaction database with real-time replication and analytics.

---

## 1. Schema Definition

```sql
-- Customers Table
CREATE TABLE Customers (
    customer_id INT PRIMARY KEY,
    age INT,
    gender VARCHAR(10),
    income DECIMAL(10,2),
    loyalty_level VARCHAR(20)
);

-- Stores Table
CREATE TABLE Stores (
    store_id INT PRIMARY KEY,
    location VARCHAR(100)
);

-- Suppliers Table
CREATE TABLE Suppliers (
    supplier_id INT PRIMARY KEY,
    lead_time INT
);

-- Products Table
CREATE TABLE Products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    unit_price DECIMAL(10,2),
    supplier_id INT,
    FOREIGN KEY (supplier_id) REFERENCES Suppliers(supplier_id)
);

-- Inventory Table
CREATE TABLE Inventory (
    store_id INT,
    product_id INT,
    inventory_level INT,
    reorder_point INT,
    reorder_quantity INT,
    PRIMARY KEY (store_id, product_id),
    FOREIGN KEY (store_id) REFERENCES Stores(store_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);

-- Weather Table
CREATE TABLE Weather (
    weather_id SERIAL PRIMARY KEY,
    weather_conditions VARCHAR(50)
);

-- Promotions Table
CREATE TABLE Promotions (
    promotion_id SERIAL PRIMARY KEY,
    promotion_type VARCHAR(50)
);

-- PaymentMethods Table
CREATE TABLE PaymentMethods (
    method_id SERIAL PRIMARY KEY,
    method_name VARCHAR(50) UNIQUE
);
```

---

## 2. Fact and Bridge Tables

```sql
-- Transactions Table
CREATE TABLE Transactions (
    transaction_id INT PRIMARY KEY,
    transaction_date TIMESTAMP,
    customer_id INT,
    store_id INT,
    payment_method_id INT,
    promotion_applied BOOLEAN,
    promotion_id INT,
    weather_id INT,
    stockout BOOLEAN,
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id),
    FOREIGN KEY (store_id) REFERENCES Stores(store_id),
    FOREIGN KEY (payment_method_id) REFERENCES PaymentMethods(method_id),
    FOREIGN KEY (promotion_id) REFERENCES Promotions(promotion_id),
    FOREIGN KEY (weather_id) REFERENCES Weather(weather_id)
);

-- TransactionDetails Table
CREATE TABLE TransactionDetails (
    transaction_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (transaction_id, product_id),
    FOREIGN KEY (transaction_id) REFERENCES Transactions(transaction_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);

-- PromotionApplications Table
CREATE TABLE PromotionApplications (
    transaction_id INT,
    promotion_id INT,
    PRIMARY KEY (transaction_id, promotion_id),
    FOREIGN KEY (transaction_id) REFERENCES Transactions(transaction_id),
    FOREIGN KEY (promotion_id) REFERENCES Promotions(promotion_id)
);

-- DemandForecast Table
CREATE TABLE DemandForecast (
    forecast_date DATE,
    store_id INT,
    product_id INT,
    forecasted_demand INT,
    actual_demand INT,
    PRIMARY KEY (forecast_date, store_id, product_id),
    FOREIGN KEY (store_id) REFERENCES Stores(store_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```

---

## 3. Trigger for Inventory Update

```sql
-- Trigger function
CREATE OR REPLACE FUNCTION update_inventory_after_detail()
RETURNS TRIGGER AS $$
DECLARE
  store INT;
BEGIN
  SELECT store_id INTO store FROM Transactions WHERE transaction_id = NEW.transaction_id;

  UPDATE Inventory
  SET inventory_level = inventory_level - NEW.quantity
  WHERE store_id = store AND product_id = NEW.product_id;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger binding
CREATE TRIGGER trg_update_inventory_after_detail
AFTER INSERT ON TransactionDetails
FOR EACH ROW
EXECUTE FUNCTION update_inventory_after_detail();
```

---

## 4. Stored Procedure for Inserting Transactions

```sql
CREATE OR REPLACE PROCEDURE insert_transaction(
    p_transaction_id INT,
    p_transaction_date TIMESTAMP,
    p_customer_id INT,
    p_store_id INT,
    p_payment_method_id INT,
    p_promotion_applied BOOLEAN,
    p_promotion_id INT,
    p_weather_id INT,
    p_stockout BOOLEAN,
    p_product_ids INT[],
    p_quantities INT[]
)
LANGUAGE plpgsql
AS $$
DECLARE
    i INT;
BEGIN
    INSERT INTO Transactions (
        transaction_id, transaction_date, customer_id, store_id,
        payment_method_id, promotion_applied, promotion_id,
        weather_id, stockout
    ) VALUES (
        p_transaction_id, p_transaction_date, p_customer_id, p_store_id,
        p_payment_method_id, p_promotion_applied, p_promotion_id,
        p_weather_id, p_stockout
    );

    FOR i IN 1 .. array_length(p_product_ids, 1) LOOP
        INSERT INTO TransactionDetails (
            transaction_id, product_id, quantity
        ) VALUES (
            p_transaction_id, p_product_ids[i], p_quantities[i]
        );
    END LOOP;
END;
$$;
```

---

## 5. Row-Level Security (RLS) Implementation

Row-Level Security ensures that each tenant (store) can only access its own data.

```sql
ALTER TABLE Transactions ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_policy ON Transactions
USING (store_id = current_setting('app.current_store')::INT);

ALTER TABLE Transactions FORCE ROW LEVEL SECURITY;
```

Before any query session, the current store context must be set:

```sql
SET app.current_store = '5'; -- Example: store_id = 5
```

---

## 6. Batch ETL Strategy for Static Tables

- Batch ETL was performed for tables like `Products`, `Customers`, `Stores`, `Promotions`, `PaymentMethods`, `Weather`, etc.
- Python scripts (using `psycopg2` and `pymongo`) were used to extract data from PostgreSQL and load it into MongoDB.
- This initial load ensures MongoDB has complete static and dimension data before real-time replication starts.

---

## 7. Kafka + MongoDB Real-Time Streaming Strategy

- Kafka Connect and Debezium were configured to monitor PostgreSQL changes.
- Only `Transactions` and `Inventory` tables were set up for real-time CDC replication.
- The changes are streamed into Kafka topics like `walmartdb.public.transactions`.
- MongoDB Sink Connector listens to these topics and writes changes to the respective MongoDB collections.

**Data Flow:**
```
PostgreSQL (CDC via Debezium) → Kafka Topics → MongoDB Sink Connector → MongoDB Collections
```

This ensures that MongoDB always has up-to-date transactional data for reporting and analytics.

---

## 8. System Architecture Diagram

```
PostgreSQL (OLTP + RLS)
│
├── Batch ETL → MongoDB (Static Tables)
│
└── Kafka + Debezium (CDC for Transactions & Inventory)
        ↓
   Kafka Topics
        ↓
MongoDB Sink Connector
        ↓
MongoDB (Real-Time Collections)
