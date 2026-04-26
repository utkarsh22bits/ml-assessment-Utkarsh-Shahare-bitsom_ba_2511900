# Part B: Business Case Analysis
## Promotion Effectiveness at a Fashion Retail Chain

---

## B1. Problem Formulation 

### (a) Machine Learning Problem Statement 

**This is a supervised regression problem.**

- **Target variable:** `items_sold` (continuous numerical value representing sales volume)
- **Input features:** Store attributes (size, location), promotion details (type, timing), temporal features (month, day of week, festival flags), and competition metrics
- **Problem type:** Regression - we aim to predict the number of items sold, which is a continuous outcome

**Justification:** We have labeled historical data with known sales outcomes, making this supervised learning. Since items_sold is continuous (not categorical), regression is the appropriate ML task type.

---

### (b) Target Variable Selection 

**Why items_sold is more reliable than total sales revenue:**

Using **sales volume (items_sold)** as the target is superior to revenue because:

1. **Price variations:** Revenue is confounded by pricing strategies that vary across promotions, seasons, and store locations. A high-revenue promotion might simply reflect higher prices rather than customer engagement.

2. **Promotion effectiveness measurement:** Items sold directly measures how many customers were attracted and how much product moved, which is the core goal of promotional campaigns.

3. **Broader applicability:** The model can generalize better across different product categories and price points. A volume-based model focuses on customer behavior patterns rather than pricing artifacts.

**This aligns with the broader ML principle:** Choose target variables that directly measure the phenomenon of interest (customer response) rather than derivative metrics influenced by external factors (pricing).

---

### (c) Modeling Strategy for Multi-Location Stores 

**Proposed alternative: Store-specific or hierarchical modeling**

A junior analyst's suggestion of a single global model has limitations:

**Problem:** Stores in different locations respond differently to promotions due to:
- Demographic differences (urban vs. rural customer preferences)
- Local competition levels
- Regional cultural factors affecting shopping patterns

**Better approach:**
1. **Option 1 - Separate models:** Train location-specific models (urban, semi-urban, rural) to capture unique response patterns
2. **Option 2 - Hierarchical/mixed-effects model:** Include store location as a feature but allow the model to learn location-specific coefficients

**Justification:** This accounts for heterogeneity across stores. Stores in rural areas may respond better to BOGO promotions, while urban stores might prefer loyalty points. A single model forces one-size-fits-all predictions that underperform in specific contexts.

---

## B2. Data and EDA Strategy 

### (a) Data Integration Strategy 

**The raw data arrives in four separate tables:**
1. **Transactions table:** transaction_date, store_id, items_sold
2. **Store attributes table:** store_id, location_type, store_size, competition_density
3. **Promotion details table:** transaction_date, store_id, promotion_type
4. **Calendar table:** date, is_weekend, is_festival flags

**Integration approach:**

**Step 1:** Join transactions with store attributes
- **Join key:** `store_id`
- **Type:** INNER JOIN (we only analyze stores with complete attribute data)

**Step 2:** Join with promotion details
- **Join keys:** `store_id` AND `transaction_date`
- **Type:** LEFT JOIN (some transactions may have no promotion)

**Step 3:** Join with calendar table
- **Join key:** `transaction_date`
- **Type:** INNER JOIN (every date should exist in calendar)

**Grain of final dataset:** One row per transaction (store-date combination)

**Aggregations before modeling:**
- If multiple promotions ran simultaneously at one store: flag as "Multi-promo" category
- Check for duplicate transaction records: keep the one with higher items_sold or aggregate
- Verify no missing store_id or transaction_date (these are critical keys)

---

### (b) EDA Strategy 

**Four key analyses to perform before building the model:**

1. **Promotion effectiveness comparison**
   - **Chart type:** Boxplot or violin plot
   - **What to look for:** Distribution of items_sold across each promotion_type (Flat Discount, BOGO, Free Gift, Category-Specific, Loyalty Points)
   - **Impact on modeling:** Identifies which promotions drive higher sales variance; may suggest interaction terms with store location

2. **Temporal sales patterns**
   - **Chart type:** Line plot of average items_sold by month and day_of_week
   - **What to look for:** Seasonality (holiday peaks), weekday vs weekend differences
   - **Impact on modeling:** Justifies temporal feature engineering (month, day_of_week); may require polynomial or seasonal terms

3. **Correlation heatmap**
   - **What to look for:** Multicollinearity between features (e.g., store_size and competition_density might be correlated)
   - **Impact on modeling:** Identifies features to drop or combine; prevents unstable coefficients in linear models

4. **Store location stratification**
   - **Chart type:** Grouped bar chart showing average items_sold by location_type and promotion_type
   - **What to look for:** Interaction effects (e.g., urban stores respond better to loyalty points, rural stores to BOGO)
   - **Impact on modeling:** Suggests interaction features or separate models per location type

---

### (c) Data Imbalance Issue 

**Problem:** 80% of transactions had no promotion active.

**Impact on the model:**
- The model will be biased toward predicting "no-promotion" baseline sales
- Promotion effect coefficients will have high variance due to limited positive samples
- The model may underestimate promotional lift

**Steps to address:**

1. **Stratified sampling:** Ensure the train-test split maintains the 80-20 imbalance ratio to avoid unrealistic test performance

2. **Feature engineering:** Create a binary flag `has_promotion` and interaction terms `promotion_type * has_promotion`

3. **Weighted loss function:** In models like Random Forest, assign higher sample weights to promotional transactions to prevent the majority class from dominating

4. **Separate analysis:** Build one model for baseline sales (no promotion) and another for promotional uplift, then combine predictions

**Why this matters:** Without addressing imbalance, the model might predict that promotions have minimal effect simply because non-promotion data dominates the training signal.

---

## B3. Model Evaluation and Deployment 

### (a) Train-Test Split Strategy 

**Recommended approach: Temporal train-test split**

**How to split:**
- Sort all transactions by `transaction_date`
- Use earliest 80% for training (e.g., Jan 2020 - Aug 2022)
- Use most recent 20% for testing (e.g., Sep 2022 - Dec 2022)

**Why random split is inappropriate:**

1. **Data leakage:** Random split would place future transactions in the training set and past transactions in the test set. The model would "know" future trends when predicting the past, inflating performance metrics artificially.

2. **Deployment realism:** In production, the model will always predict future sales based only on historical data. A random split doesn't simulate this constraint.

3. **Temporal dependencies:** Sales data has autocorrelation (this month's sales correlate with last month's). Random split breaks these dependencies.

**Evaluation metrics:**
- **RMSE (Root Mean Squared Error):** Penalizes large prediction errors heavily; useful for identifying cases where the model badly misses
- **MAE (Mean Absolute Error):** More interpretable (average error in items sold); less sensitive to outliers

Both metrics should be reported on the test set (most recent 20% of data) to assess true future performance.

---

### (b) Model Recommendation Explanation

**Scenario:** After training, the model recommends:
- **Store 12:** Loyalty Points Bonus (December)
- **Store 12:** Flat Discount (March)

**Why different recommendations for the same store in different months:**

1. **Seasonal shopping behavior:**
   - **December:** Holiday season - customers already plan large purchases for gifts. Loyalty points encourage repeat visits and basket-building across multiple trips, maximizing customer lifetime value during peak season.
   - **March:** Post-holiday lull - customers are price-sensitive after holiday spending. Flat discounts provide immediate savings that drive foot traffic when engagement naturally dips.

2. **Feature importance captured by the model:**
   - The model learned interaction effects between `promotion_type`, `month`, and `store_id`
   - December has high `is_festival=1` flags, which the model associates with loyalty-driven behavior
   - March has no major festivals; the model defaults to price-driven promotions

3. **Communicating to the marketing team:**
   - "The model identified that Store 12's December customers respond better to loyalty incentives because they're already committed shoppers. In March, when foot traffic drops, a direct discount is needed to compete for price-sensitive customers."
   - Show data: Compare historical sales for Store 12 in Dec (with Loyalty Points) vs. March (with Flat Discount) to validate the recommendation.

---

### (c) Deployment and Monitoring Strategy 

**End-to-end deployment process:**

**Step 1: Model persistence**
- Save the trained pipeline (preprocessing + model) using `joblib` or `pickle`
- Version control: Tag model with training date and performance metrics

**Step 2: Monthly batch prediction**
- Input: All 50 stores × 5 promotion types = 250 scenarios for next month
- Run predictions for each (store_id, promotion_type) combination
- Prepare inputs: Fetch next month's calendar features (is_weekend, is_festival), store attributes from database

**Step 3: Recommendation generation**
- For each store, select the promotion_type with highest predicted items_sold
- Output: CSV file with columns [store_id, recommended_promotion, predicted_items_sold, confidence_interval]

**Step 4: Delivery to marketing team**
- Upload recommendations to shared dashboard
- Include visualizations: bar chart showing predicted lift for each store, heatmap of recommendations across stores

**Monitoring plan:**

1. **Performance tracking:**
   - Each month, compare actual items_sold (after promotion is deployed) against predictions
   - Calculate rolling MAE and RMSE over recent months
   - Alert if error increases by >20% (indicates model degradation)

2. **Retraining triggers:**
   - **Scheduled:** Retrain quarterly with latest 3 years of data to capture evolving trends
   - **Performance-based:** If test MAE degrades by >15%, trigger immediate retraining
   - **Data drift:** Monitor feature distributions (e.g., if competition_density changes significantly, retrain)

3. **What to feed back into retraining:**
   - New monthly data (store transactions from deployed promotions)
   - Updated calendar (new festivals, holidays)
   - Store attribute changes (if stores remodel or location_type shifts)

**Why retraining matters:** Customer preferences and competitive landscapes shift over time. A model trained in 2020 may not capture 2023 shopping behaviors (e.g., increased online competition, inflation effects). Monthly monitoring with quarterly retraining ensures the model stays relevant.

