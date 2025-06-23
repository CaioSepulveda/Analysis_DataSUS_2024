# DATASUS Procurement Efficiency Dashboard

## Table of Contents
1.  [Project Overview](#1-project-overview)
2.  [Data Source](#2-data-source)
3.  [Data Preparation & Cleaning](#3-data-preparation--cleaning)
    * [Initial Data Loading & Basic Cleaning (Python/Pandas)](#initial-data-loading--basic-cleaning-pythonpandas)
    * [Data Refinement & Feature Engineering (BigQuery SQL)](#data-refinement--feature-engineering-bigquery-sql)
4.  [Key Metrics & Analysis](#4-key-metrics--analysis)
    * [Excess Spending Calculation](#excess-spending-calculation)
    * [Supplier Competitiveness Ranking](#supplier-competitiveness-ranking)
5.  [Dashboard Overview (Looker Studio)](#5-dashboard-overview-looker-studio)
    * [Page 1: Procurement Efficiency Report](#page-1-procurement-efficiency-report)
    * [Page 2: Best Supplier Lookup](#page-2-best-supplier-lookup)
6.  [Future Enhancements](#6-future-enhancements)

---

## 1. Project Overview

This project focuses on analyzing public procurement data from DATASUS (Department of Informatics of the Unified Health System in Brazil) for the year 2024. The primary goal is to enhance **resource efficiency** by identifying and visualizing **excess spending** in medication and supply purchases. Its part of the [#DataGlowUP45](https://heitorsasaki.substack.com/p/data-glow-up-45) challenge

The project involves a multi-stage data cleaning and transformation process, followed by the creation of an interactive dashboard in Looker Studio. The dashboard provides insights into:
* Overall excess spending and its proportion to total expenditure.
* Key drivers of excess spending (by medication, purchase modality, and state).
* A dedicated tool for users to quickly find the most competitive supplier for any given medication.

## 2. Data Source

The raw data is provided in a CSV file:
* `2024.csv`: Contains detailed records of procurement transactions, including institution names, CNPJs, item descriptions, quantities, unit prices, suppliers, and purchase modalities.

## 3. Data Preparation & Cleaning

The raw data underwent several cleaning and transformation steps to ensure data quality and prepare it for analysis and visualization.

### Initial Data Loading & Basic Cleaning (Python/Pandas)

The `2024.csv` file was loaded into a Pandas DataFrame in a Google Colab environment.

* **Encoding & Delimiter:** `encoding='latin-1'` and `sep=';'` were used to correctly read the file due to character encoding and delimiter issues. `on_bad_lines='skip'` was applied to handle malformed rows.
* **Missing Values Handling:**
    * The `ANVISA` column (over 60% missing values) was **dropped**.
    * Rows with missing values in `Nome Instituição` (0.04%) and `Unidade Fornecimento` (0.06%) were **dropped** due to their low percentage.
    * Missing values in the `Inserção` column (around 14%) were **imputed** using the median (if numeric) or mode (if non-numeric, though it was later converted to datetime).
* **Data Type Conversion:**
    * `Preço Unitário` and `Preço Total` were converted from `object` to `float64` after replacing commas (`,`) with periods (`.`) as decimal separators.
    * `Compra` and `Inserção` (which were numeric OLE Automation Dates) were converted to `datetime64[ns]` objects, using `origin='1899-12-30'` and `unit='D'`.
* **Output:** The partially cleaned data was saved as `2024_clean.csv`.

### Data Refinement & Feature Engineering (BigQuery SQL)

Further cleaning and feature engineering were performed directly in Google BigQuery, building upon the `2024_clean.csv` data (assumed to be loaded into `data-glow-up-45.datasus_2024.2024_clean`).

**3.1. Cleaning `Descrição CATMAT` and Creating `2024_clean_refined`**

This step creates a new table `2024_clean_refined` by cleaning the `Descrição CATMAT` column and renaming it to `Medicamento` for easier querying.

* **Purpose:** The original `Descrição CATMAT` contained HTML entities (e.g., `&#211;` for `Ó`), HTML tags (`<DIV>`, `<SPAN>`), and excessive whitespace. This made the names difficult to read, unreliable for analysis (grouping, distinct counts), and problematic for comparisons. The `Medicamento` column provides a clean, standardized, and easy-to-use name.
* **Transformations:**
    * Replaces common HTML entities (e.g., `&#211;` to `Ó`, `&#205;` to `Í`, `&#193;` to `Á`, `&#201;` to `É`, `&#195;` to `Ã`).
    * Replaces non-breaking spaces (`&#160;`) with regular spaces.
    * Removes all HTML tags (`<[^>]+>`).
    * Collapses multiple whitespace characters into single spaces (`\s+`).
    * Trims leading and trailing spaces (`TRIM`).
* **Output:** `data-glow-up-45.datasus_2024.2024_clean_refined` table.

**3.2. Creating `2024_final_dashboard` Table**

This table is the primary data source for the main dashboard page, containing calculated metrics for "excess spending".

* **Purpose:** To pre-calculate key metrics related to excess spending, allowing for faster dashboard performance and consistent metric definitions.
* **Calculations:**
    * **Quartiles (Q1, Q2, Q3):** `PERCENTILE_CONT` is used to calculate the 25th, 50th (median), and 75th percentiles for `Qtd Itens Comprados` and `Preço Unitário`, partitioned by `Medicamento`.
    * **Upper Bounds:** `qtd_upper_bound` and `price_upper_bound` are calculated using the 1.5 * IQR (Interquartile Range) method, a common statistical approach to identify potential outliers.
    * **`Gasto_Em_Excesso` (Excess Spending):** Defined for items where both `Preço Unitário` is greater than `price_upper_bound` AND `Qtd Itens Comprados` is greater than `qtd_upper_bound`. The excess is quantified as `(Qtd Itens Comprados - qtd_Q2) * (Preço Unitário - price_Q2)`. (Note: This is a calculated magnitude of excess, not necessarily a direct monetary amount over a baseline).
    * **`Possui_Excesso` (Has Excess):** A boolean flag ('Sim'/'Não') indicating if an item meets the `Gasto_Em_Excesso` criteria.
* **Column Renaming:** Several columns are renamed for clarity in the dashboard (e.g., `Preco_Total`, `Modalidade_De_Compra`, `Preco_Unitario`, `Qtd_Itens_Comprados`).
* **Output:** `data-glow-up-45.datasus_2024.2024_final_dashboard` table.

**3.3. Creating `Suppliers_Ranking` Table**

This table supports the "Best Supplier Lookup" functionality on the second dashboard page.

* **Purpose:** To pre-compute the best (lowest average price) supplier for each unique medication.
* **Calculations:**
    * **`Preco_Unitario_Medio`:** Calculates the average unit price for each `Medicamento` and `Fornecedor` combination.
    * **`Rank`:** Uses `ROW_NUMBER()` partitioned by `Medicamento` and ordered by `Preco_Unitario_Medio` (ASC) to assign a rank. Rank 1 indicates the best supplier (lowest average price). `ROW_NUMBER()` ensures a single best supplier is identified even in case of price ties.
* **Filtering:** Filters for `Rank = 1`.
* **Output:** `data-glow-up-45.datasus_2024.Suppliers_Ranking` table.

## 4. Key Metrics & Analysis

### Excess Spending Calculation

The core of the "resource efficiency" analysis relies on the `Gasto_Em_Excesso` metric. This metric identifies purchases where both the unit price and the quantity purchased exceed outlier thresholds (defined by 1.5 * IQR above Q3). The value represents a combined magnitude of these two factors being "excessive."

### Supplier Competitiveness Ranking

Suppliers are ranked by their average unit price for each specific medication, allowing for quick identification of the most cost-effective provider per item.

## 5. Dashboard Overview (Looker Studio)

The Looker Studio dashboard is designed with two main pages to provide comprehensive insights and utility. You can see it clicking [Here](https://lookerstudio.google.com/reporting/113221bc-cc17-4a99-b9b7-fcf0e0e6ac0e)

### Page 1: Procurement Efficiency Report

* **Purpose:** Provide an overview of excess spending, its impact, and main drivers.
* **Key Visuals:**
    * **Scorecards:**
        * `Orçamento Executado` (Total Budget Spent): Provides context for overall spending.
        * `Excesso Total`: Sum of `Gasto_Em_Excesso`.
        * `% em Excesso`: Proportion of `Excesso Total` relative to `Orçamento Executado`.
        * `Total de Fornecedores` & `Medicamentos/Insumos`: Overall summary counts from the dataset.
    * **Bar Chart: `Gasto em Excesso por Modalidade de Compra`:** Stacked bar chart showing `Gasto_Em_Excesso` vs. `Orçamento Executado` for each purchase modality. Uses **orange** for `Gasto_Em_Excesso` (highlight/attention) and **blue** for `Orçamento Executado` (normal/total).
    * **Bar Chart: `Medicamentos com maior Gasto em Excesso`:** Horizontal bar chart displaying the top medications by `Gasto_Em_Excesso`. Uses the **orange** color for consistency with "excess" representation.
* **Filters:**
    * **Consolidated Filter Panel (Left Side):**
        * `Medicamento` (Text search input): Allows global filtering by medication name.
        * `UF` (Dropdown): Filters by state.
        * `Filtrar por Impacto do Excesso` (Slider Control): Filters data based on the calculated `Gasto_Em_Excesso` magnitude, allowing users to focus on higher impact items.
    * **Date Range Control:** Located near the consolidated filter panel for consistent user experience, allowing filtering by purchase date range (e.g., month, quarter, year).
* **Interactivity:** Cross-filtering is enabled across charts to allow dynamic exploration (e.g., clicking a state in a map/bar chart filters all other visuals for that state).

### Page 2: Best Supplier Lookup

* **Purpose:** Provide a quick, intuitive lookup tool for users to find the most competitive supplier for a specific medication.
* **Key Visuals:**
    * **Clear Title & Instructions:** "Melhor Fornecedor por medicamento/insumo" and "Pesquise aqui o Medicamento/Insumo desejado" provide immediate guidance.
    * **Interactive Table Chart:**
        * **Columns:** `Medicamento` (cleaned name), `Fornecedor`, `Preço Unitário Médio`.
        * **Search Bar:** Automatically included with the table chart, allowing users to type medication names directly.
        * **Sorting & Pagination:** Enabled by default.
* **Layout:** Focused and minimalist, emphasizing the search bar and the table results.

## 6  Future Enhancements

* **Advanced Outlier Detection:** Explore more sophisticated statistical methods for outlier identification (e.g., Z-scores, Isolation Forest) if `1.5 * IQR` is insufficient.
* **Categorization of Medications:** Implement a system to categorize `Medicamento` (e.g., Antibiotics, Analgesics) to enable higher-level analysis.
* **Supplier Performance Over Time:** Develop visuals to track supplier performance and excess spending trends over longer periods, if historical data becomes available.
* **Automated Data Pipeline:** Set up automated ETL (Extract, Transform, Load) processes using tools like Cloud Composer (Apache Airflow) or Dataflow to regularly update the BigQuery tables.
* **User Feedback Integration:** Implement a mechanism to collect user feedback on the dashboard's utility and design.
