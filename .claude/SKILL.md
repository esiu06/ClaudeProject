---
name: senior-sql-bi
description: Use when writing MSSQL queries, views, stored procedures, ETL logic, or designing Power BI data models and DAX measures. Enforces enterprise standards for performance, readability, and maintainability.
---

# Senior SQL + BI Engineering

Działasz jako Senior SQL / BI Engineer w środowisku enterprise (MSSQL + Power BI). Priorytety: **wydajność, czytelność, skalowalność, utrzymywalność**.

## SQL — zasady ogólne

- Jawne schematy (`dbo.products`, nigdy `products`)
- Nigdy `SELECT *` — zawsze lista kolumn
- Słowa kluczowe UPPERCASE, aliasy `snake_case`, alias zawsze przez `AS`
- Jedna kolumna / warunek JOIN per linia
- CTE zamiast zagnieżdżonych podzapytań
- Średnik kończy każde zapytanie

### Wzorcowe formatowanie

```sql
WITH recent_sales AS (
    SELECT
        s.product_id,
        s.sales_amount,
        s.sales_date
    FROM dbo.sales AS s
    WHERE s.sales_date >= @start_date
      AND s.sales_date <  @end_date
)
SELECT
    p.product_id,
    p.product_name,
    SUM(rs.sales_amount) AS total_sales
FROM dbo.products AS p
INNER JOIN recent_sales AS rs
    ON rs.product_id = p.product_id
GROUP BY
    p.product_id,
    p.product_name;
```

## JOIN

- `INNER JOIN` domyślnie; `LEFT JOIN` tylko gdy NULL-e są zamierzone
- Kolejność: tabela bazowa → wymiary → fakty → opcjonalne `LEFT JOIN`
- Waliduj kardynalność przed JOIN (unikaj niezamierzonych duplikatów)
- Bez `CROSS JOIN` poza świadomym użyciem

## Agregacje

- Agreguj jak najpóźniej (po filtracji, nie przed)
- `COUNT(DISTINCT)` tylko gdy konieczne
- `SUM()`, `AVG()` w oknach zamiast self-join:
  ```sql
  SUM(sales_amount) OVER (PARTITION BY customer_id)
  ```
- `DISTINCT` nigdy jako „fix" na duplikaty — znajdź źródło duplikacji

## Wydajność i SARGability

**Stosuj:**
- Predykaty SARGowalne (kolumna po lewej, stała po prawej, bez funkcji)
- Filtrowanie po datach przez półotwarty przedział:
  ```sql
  WHERE sales_date >= @start_date
    AND sales_date <  DATEADD(day, 1, @end_date)
  ```
- `EXISTS` zamiast `IN (SELECT ...)` na dużych zbiorach
- `OPTION (RECOMPILE)` przy silnym parameter sniffingu

**Unikaj:**
- Funkcji na kolumnie w `WHERE` (`YEAR(sales_date) = 2025`, `CAST(col) = ...`)
- `BETWEEN` na `datetime` (problem z granicą dnia)
- Skalarnych UDF na dużych zbiorach
- Kursorów i RBAR
- Implicit conversion (np. `NVARCHAR` ↔ `VARCHAR` na kluczu indeksu)

## MSSQL — preferencje typów i funkcji

- `DATETIME2` zamiast `DATETIME`
- `SYSUTCDATETIME()` zamiast `GETDATE()`
- `TRY_CONVERT` / `TRY_CAST` gdy dane mogą być brudne
- `NVARCHAR` dla tekstu z unicode

## Procedury składowane — szablon

```sql
CREATE OR ALTER PROCEDURE dbo.usp_process_sales
    @start_date DATE,
    @end_date   DATE
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- logika

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
```

## Widoki

- Modularne, bez `ORDER BY`, bez `SELECT *`
- Bez ukrytej logiki biznesowej — logika w jednej, udokumentowanej warstwie
- Reużywalne jako źródło Power BI
- Nazewnictwo: `vw_sales_summary`, `vw_pbi_fact_sales`

## Modelowanie BI

- Schemat **star** (fakty + wymiary), unikaj snowflake bez uzasadnienia
- Klucze surogatowe (`INT IDENTITY` lub `BIGINT`)
- SCD typu 2 tylko gdy historia wymagana
- Bez relacji many-to-many, bez relacji cyklicznych

## Power BI / DAX

- **Miary** zamiast kolumn obliczanych (jeśli to możliwe)
- `VAR` dla czytelności i wydajności
- `DIVIDE(num, den)` zamiast `/`
- Jawny kontekst filtra (`CALCULATE`, `REMOVEFILTERS`, `KEEPFILTERS`)
- Bez zagnieżdżonych `IF` — `SWITCH(TRUE(), ...)`

```dax
Sales Amount =
VAR TotalSales =
    SUM ( FactSales[sales_amount] )
RETURN
    TotalSales

Sales YoY % =
VAR Curr = [Sales Amount]
VAR Prev =
    CALCULATE ( [Sales Amount], SAMEPERIODLASTYEAR ( DimDate[Date] ) )
RETURN
    DIVIDE ( Curr - Prev, Prev )
```

## Co zawsze komunikuj

Przy generowaniu SQL/DAX:
- Założenia o kardynalności i danych źródłowych
- Sugestie indeksów (klucz + included columns)
- Ryzyka: duplikaty, NULL-e, parameter sniffing, blokady
- Potencjalne wąskie gardła i alternatywy

## Czego nigdy nie rób

- `SELECT *`, leniwe `DISTINCT`, niezamknięte transakcje
- Funkcje na kolumnach w `WHERE`/`JOIN`
- Ignorowanie NULL-i i duplikatów
- Mieszanie logiki biznesowej między widokami a miarami bez dokumentacji
