
### End-to-End Pipeline Flow
Source Data → EDA → Mart Layer → Modeling → Feature Engineering → Evaluation

![End-to-End Pipeline](Images/Flow.png)

---

## 1. Data Understanding & Methodology
### 1.1 Source Data
- This project uses a multi-table e-commerce dataset containing customer, order, product, payment, review, seller, and geolocation information to analyze purchasing behavior and predict customer loyalty.

### 1.2 Objective 
- The data is used to predict whether a first-time customer will become a repeat buyer and to estimate the expected order value based on their initial purchase behavior.

### 1.3 Exploratory Data Analysis (EDA)

**Nulls & Basic Transformations**  
- Null value checks and basic data cleaning were performed across all tables to ensure schema consistency and reliable downstream joins. Missing values were analyzed.

**Repeat Customer Analysis**  
- Customer behavior was analyzed using `customer_unique_id` as the true customer identifier, enabling clear separation between first-time and repeat buyers. Purchase timelines were examined to understand ordering frequency and customer retention patterns.

**Class Imbalance Insight**  
- EDA revealed a severe class imbalance, with only ~3% of customers making repeat purchases. This insight informed model evaluation strategy and metric selection, with emphasis on PR-AUC over accuracy.

**Data Modeling Implications**  
- Findings from EDA directly guided mart design and feature engineering, ensuring consistent grain alignment across datasets and preventing data leakage during model training.

---

## 3. Data Modeling (Mart & Schema Design)

### 3.1 Grain Alignment Strategy

**1. Identifying Grain Mismatches**  
- During data exploration, we identified that different source tables existed at different levels of granularity (e.g., order-level, product-level, seller-level, and review-level). Directly joining these tables would have resulted in record duplication and inflated metrics.

**2. Aggregation & Roll-Up Validation**  
- To address this, each table was pre-aggregated to a common grain (primarily `order_id`) using appropriate roll-up logic. Aggregations were carefully validated by reconciling totals (such as item counts and order values) against source data to ensure no loss or distortion of information.

**3. Duplicate Detection & Prevention**  
- Extensive checks were performed to identify duplicate keys and unintended row multiplication after joins. This ensured that each order and customer was represented exactly once in the master dataset, preserving data integrity and preventing leakage into the modeling pipeline.

### 3.2 Mart-Level Transformations

Two key mart-level transformations were implemented to ensure data consistency and enrich modeling features.

**1. Order Items Roll-Up**  
- The `order_items` table operates at a composite grain of `order_id`, `seller_id`, and `product_id`. To enable order-level analysis, item-level records were aggregated at this composite grain to infer quantities and compute monetary metrics. These metrics were then rolled up to the `order_id` level, ensuring join-safe integration with other order-level marts.

![OrderItemsRollupLogic](EDA/OrderItems.png)

**2. Customer–Seller–Geography Enrichment**  
Customer and seller datasets contained duplicate zip code prefixes, and the geolocation table mapped a single zip code to multiple latitude–longitude pairs. To address this, a bridge table was created at the zip-level by selecting the mos

![CustomerSellerGeo](Data%20Modelling/CustomerGeoEnrichment.png)

### 3.3 Master Table Construction

- Once all upstream marts were validated and aligned to a **unique `order_id` grain**, they were integrated to form the final master table. Each mart was joined using **left joins** with the core orders table to ensure full order coverage while preserving data integrity. This approach produced a single, unified order-level dataset containing customer, product, payment, review, and geographic features, suitable for downstream analysis and modeling.

## 4. Label Construction Methodology

This section outlines the logic used to derive target variables from transactional data. By transforming raw purchase history into behavioral labels, the modeling approach focuses on predicting long-term customer value rather than isolated transactions.

### 4.1 Customer Conversion (Loyalty) Label

The conversion label identifies customers who transition from one-time buyers to repeat purchasers.

**Scope**  
This logic is applied across the entire dataset, covering both first-time and repeat customers.

**Label Definition**
- **Converted (1):** Customers with two or more unique `order_id` values across their lifetime.
- **Non-Converted (0):** Customers with only a single recorded order.

**Methodology**
Customer-level order history was analyzed using the full dataset to determine conversion status. The resulting label was then mapped back to the `order_id` grain, allowing the model to learn which characteristics of an initial purchase are indicative of future return behavior.

---

### 4.2 Order Value Label (Average Order Value – AOV)

In addition to conversion, a continuous target was defined to capture customer spending behavior.

**Scope**  
This label is applied exclusively to repeat customers.

**Label Definition**  
For each repeat customer, the **Average Order Value (AOV)** was calculated as:

\[
\text{AOV}_c = \frac{1}{N_c} \sum_{i=1}^{N_c} \text{TotalOrderValue}_{c,i}
\]


The computed AOV was assigned as the target value for all orders associated with that customer.

**Predictive Intent**  
By predicting a customer’s average spend rather than a single future transaction, the model reduces sensitivity to outlier purchases and focuses on stable spending behavior. This enables more reliable estimation of long-term revenue and customer lifetime value (LTV).

---

## 5. Modeling & Evaluation

![Scope](Images/Scope.png)

### 5.1 Machine Learning Model 1: Customer Conversion (Loyalty Prediction)

The objective of this model is to predict the probability that a customer transitions from a one-time buyer to a repeat customer based on their initial transactional and behavioral signals.(Both for Repeated and First Time Customers)

### Model Selection & Iteration

A **Champion–Challenger** approach was adopted to evaluate multiple algorithms and identify the most effective modeling strategy:

- **Baseline Model:** Logistic Regression, used to establish a performance floor and provide interpretability.
- **Tree-Based Models:** Random Forest, XGBoost, and LightGBM to capture non-linear interactions in customer behavior.
- **Final Approach:** An ensemble model combining the strongest gradient-boosted tree models to improve predictive stability and robustness.

### 5.2 Key Challenges & Mitigation Strategies

**Class Imbalance**
- The dataset exhibited severe imbalance, with repeat customers representing a small minority.
- This made it difficult for models to identify weak but meaningful loyalty signals.

**Solution**
- Extensive feature engineering was performed to amplify behavioral signals, including purchase diversity, temporal patterns, and spending characteristics.

**Model Diagnostics & Validation**
- Conducted data leakage audits to ensure no future information was present in first-order features.
- Applied scaling and skewness correction to monetary features to improve model convergence.
- Analyzed feature importance using model-based metrics and SHAP values to confirm that predictions were driven by meaningful signals.

**Noise Reduction**
- Examined feature correlations to identify and remove highly collinear variables.
- This reduced redundancy, simplified the feature space, and improved generalization to unseen data.

### 5.3 Key Modeling Insights

**Signal Amplification**  
- Raw transactional features alone were insufficient to distinguish repeat customers in a highly imbalanced dataset. Extensive feature engineering was required to amplify weak but meaningful loyalty signals.

**High-Impact Features**  
- Derived metrics such as a **Sentiment Index** (capturing customer satisfaction) and **Engagement Velocity** contributed the most significant predictive lift.

**Ratio-Based Features**  
- Tree-based models performed more effectively when using **ratio features** (e.g., value-to-volume) rather than individual raw metrics, as ratios enabled more informative decision splits.

**Model Selection Outcome**  
- XGBoost emerged as the strongest individual model due to its robustness to sparse data and missing values.

**Noise Reduction Strategy**  
- Highly correlated and redundant features were identified using a correlation matrix and removed to reduce noise and improve generalization.

**Performance Gain**  
- The combined impact of ratio-based feature engineering and noise reduction resulted in a **4× improvement in PR-AUC**, increasing performance from a ~2% baseline to **~8.3%**.


---

This model focuses on learning **behavioral patterns** that indicate long-term customer retention rather than short-term transactional noise.

### Models Evaluated
### Handling Class Imbalance
### Evaluation Metrics
### Model Performance Summary

---

## 6. Key Challenges & Design Decisions
- Grain mismatches across datasets
- Data leakage prevention
- Class imbalance handling
- Metric selection rationale

---

## 7. How to Run the Project
### Prerequisites
### Installation
### Execution Order

---

## 8. Results & Insights
- Model comparison summary
- Business interpretation of results

---

## 9. Limitations & Future Work
- Known constraints
- Planned improvements

---

## 10. Conclusion
Final takeaways and impact
