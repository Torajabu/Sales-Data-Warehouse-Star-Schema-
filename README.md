# Sales Data Warehouse (Star Schema)

A simple SQL-based ETL pipeline that transforms a flat `Orders` table into a dimensional **star schema** data warehouse — staging → transformation → dimensions → fact table. ( https://drive.google.com/file/d/14v2oHfbGUPcju18IQOVqf6_n3FjKgqoX/view?usp=drive_link )

## Architecture
![Data Flow Diagram](https://raw.githubusercontent.com/Torajabu/Sales-Data-Warehouse-Star-Schema-/main/arch.svg)

```
sales_8am.Orders
        │
        ▼
sales_8am_model.Orders_model        (raw copy)
        │
        ▼
sales_8am_dwh_model.staging_model   (staging layer)
        │
        ▼
sales_8am_dwh_model.transformation_model   (cleaned view, NULLs removed)
        │
        ├──► Dim_Customers
        ├──► DimProducts
        ├──► DimRegion
        ├──► DimOrderDate
        │
        ▼
sales_8am_dwh_model.Fact_Sales
```

## Source Schema

```sql
CREATE TABLE sales_8am.Orders (
    OrderID         INT,
    OrderDate       DATE,
    CustomerID      INT,
    CustomerName    VARCHAR(100),
    CustomerEmail   VARCHAR(100),
    ProductID       INT,
    ProductName     VARCHAR(100),
    ProductCategory VARCHAR(50),
    RegionID        INT,
    RegionName      VARCHAR(50),
    Country         VARCHAR(50),
    Quantity        INT,
    UnitPrice       DECIMAL(10,2),
    TotalAmount     DECIMAL(10,2)
);
```

## Pipeline Stages

### 1. Staging
Raw data is copied as-is into the warehouse, with no transformations, to preserve a clean source-of-truth snapshot:
- `sales_8am_model.Orders_model`
- `sales_8am_dwh_model.staging_model`

### 2. Transformation
A view (`transformation_model`) filters out records with a missing `CustomerID`, acting as a lightweight data quality gate.

### 3. Dimension Tables
Each dimension follows the same pattern: extract distinct attributes, generate a surrogate key with `ROW_NUMBER()`, then materialize into a physical table.

| Table | Natural Key | Surrogate Key |
|---|---|---|
| `Dim_Customers` | `CustomerID` | `Surr_CustomerKey` |
| `DimProducts` | `ProductID` | `Surr_ProductKey` |
| `DimRegion` | `RegionID` | `Surr_RegionKey` |
| `DimOrderDate` | `OrderDate` | `Surr_OrderDateKey` |

Surrogate keys decouple the warehouse from source-system key changes — a standard dimensional modeling practice.

### 4. Fact Table
`Fact_Sales` stores order-level measures (`Quantity`, `UnitPrice`, `TotalAmount`) joined against each dimension's surrogate key:

```sql
CREATE TABLE sales_8am_dwh_model.Fact_Sales (
    OrderID             INT,
    Surr_OrderDateKey   INT,
    Quantity            INT,
    UnitPrice           DECIMAL(10,2),
    TotalAmount         DECIMAL(10,2),
    Surr_CustomerKey    INT,
    Surr_ProductKey     INT,
    Surr_RegionKey      INT
);
```

## Usage

Run `dwh_star_schema_corrected_v2.sql` top to bottom against a database that already has `sales_8am.Orders` populated. The script will:

1. Create the model and DWH databases
2. Stage and clean the data
3. Build and populate all four dimension tables
4. Build and populate the fact table

```bash
mysql -u <user> -p < dwh_star_schema_corrected_v2.sql
```

> Adjust the client invocation above if you're running this on Postgres, SQL Server, etc. — syntax for `CREATE TABLE ... AS SELECT`, `ROW_NUMBER()`, and views is broadly compatible across engines but may need minor tweaks.

## Notes

- All joins to the fact table use `LEFT JOIN` so orders without a matching dimension row aren't dropped (their surrogate key will be `NULL`).
- The date dimension is included but optional — drop it and store `OrderDate` directly on the fact table if a low-grain date dimension isn't needed for your use case.
