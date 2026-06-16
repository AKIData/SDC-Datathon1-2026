# 🍂SDC Datathon AKI Data Process

### **Project Title:** Predicting Socio-Economic Impacts Post-Mobile Money Deployment
### **Author:** `AKI Data - Amani Koros, Karanei Kimutai, Ivan Mayabi`
### **Timeline Context:** June 2026 Datathon Track

---

## 1. Executive Summary
Welcome to the official post-mortem and operational blueprint for our predictive financial health engine. This project analyzes raw survey micro-data tracking Kenyan financial inclusion metrics (grounded in national data profiles like GeoPoll and FinAccess report patterns). 

Our assignment seemed innocent: predict an individual's changing financial stability using **28 high-quality survey columns**. What followed was a dramatic, hard-fought architectural journey where our code underwent an extreme existential pivot—evolving from an utterly blind **13% $R^2$ regression model that used 111 noisy columns** to a highly optimized, beautifully structured **3-Class Balanced Classifier** pulling an honest **52% generalized accuracy** on completely untouched test data.

---

## 2. Tools & Stack Architecture
Our technical pipeline was carefully engineered:
* **Data Wrangling & Pipeline Preservation:** `Pandas`
* **The Early Contenders (The Traps):** Scikit-Learn's `RandomForestRegressor`.
* **The Heavyweight Production Stack:** `LGBMClassifier` (LightGBM Native Boosting) and Scikit-Learn's `RandomForestClassifier`.
* **Validation Metrics Engine:** `classification_report`, `confusion_matrix`, and `ConfusionMatrixDisplay` for true diagnostic visualizations.

---

## 3. Methodology & The Great Pivots

### 🛑 Step 1: The Regression Wall ($R^2 = 0.13$)
* **The Mistake:** Because financial status changes feel numeric, we naturally assumed we should map them on a continuous scale using a `RandomForestRegressor`.
* **The Reality:** The model flatlined at an $R^2$ of **0.11 to 0.13**. Human socio-economic shifts do not follow smooth, predictable decimal slopes. People do not smoothly drift down by `0.342` units of financial health; they encounter sudden, discrete life events, structural barriers, or regional shocks. 
* **LGBRegression:** We unfortunately also built a model using `LGBMRegression` before realising that no Regression model would fit our data requirements.

### 💡 Step 2: The Multi-Class Bucket Epiphany
* **The Pivot:** We tore down the regression landscape and remapped our target column explicitly into three distinct life states:
    * **`-1` (Worsened):** Financial shock or transaction barriers eroded economic status.
    * **`0` (Stayed Same):** Maintained stable economic equilibrium.
    * **`1` (Improved):** Successfully leveraged mobile money channels or formal networks to climb.
* **The Immediate Win:** The moment we initialized an `LGBMClassifier` on these explicit target buckets, our validation accuracy immediately leaped to **57%**!

### ✂️ Step 3: Killing the One-Hot Column Bloat (From 111 to 27 Pillars)
* **The Issue:** Our initial preprocessing transformed categories into a massive, sparse grid of **88 columns for use in regressors**. Tree models were constantly getting lost splitting on endless artificial `0` or `1` branches.
* **The Clean Up:** We aggressively grouped high-cardinality flags. Counties were mapped into high-level geographic regions; household sizes were compressed into three bands (`1-6`, `6-11`, `>11`); and financial products were binned. By utilizing LightGBM's native categorical engine, we dropped our column count to **27 clean, native features**. The model ran 3x faster without sacrificing a single point of intelligence.
* **The benefit of starting with regression:** We realised however that we were able to combat a lot of the noise from the initial data set. Because `Random Forest Regression made each unique column value post the initial clean a column on its own`, we were able to target and group individual values into rows. This gave us the intuition to clean up as mentioned above. We were able to do this until the `data that was left was impossible to drop without removing useful data points`.

### ⚖️ Step 4: The Honest Balancer (`class_weight='balanced'`)
* **The Realization:** Our 57% accuracy model was cheating! Because the data was dominated by people who "Worsened (-1)", the model guessed `-1` almost every time, leaving the recall of our smaller groups trapped below 25%.
* **The Treatment:** We turned on automated class balancing. Our overall accuracy settled at a perfectly stable and honest **52%**, but our recall for the "Improved" bucket doubled to **56%**—meaning the model was now genuinely catching over half of the true upward socio-economic climbers.

### Step 5: (0.7-0.15-0.15 Split) Validation Tests on all Models
* **All models used:** We tested various algorithms to attempt to come to the best solution and these were the results of each when compared against our 0.15% validation data:
    * Regression
        * `Random Forest Regression` = r^2 of `0.13`
        * `Light GBM Regression` = r^2 of `0.13`
    * Classification
        * `Light GBM Classification` = Mean f1_score of `0.52`
        * `CatBoost Classifier` = Mean f1_score of `0.52`
        * `HistGradientBoosting Classifier` = Mean f1_score of `0.52`
        * `Random Forest Classifier` = Mean f1_score of `0.53`

---

## 4. Final Performance Report (The Moment of Truth)
After debugging a silent string-to-integer misalignment on our final test set split  our champion **Random Forest Classifier** yielded a beautifully robust, un-overfitted test report card: This was computed on the final 0.15% test data.

```text
===Corrected Random Forest Classifier Test Performance ===
Overall Accuracy: 0.52

              precision    recall  f1-score   support

          -1       0.68      0.57      0.62      1638
           0       0.41      0.39      0.40       831
           1       0.37      0.56      0.44       659

    accuracy                           0.52      3128
   macro avg       0.49      0.51      0.49      3128
weighted avg       0.54      0.52      0.52      3128
```

## 5. Recommendations

#### With all due respect, the model is `20% better than the statistical average`.


#### Bias in the model

* The model is likely to predict worsened people as worsened
* The model is likely to predict improved people as improved
* The model is likely to predict people who stayed the same as worsened, proving it is negatively biased.

* As such, if it is to be used by **financial institutions** to `govern borrowing or issuing of loans`, then it is better fitted to check financial status
* However if given to **NGOs or Government**, it is likely to **discriminate in favour people who have stayed the same** saying they have worsened, as such it may lead to `giving more people help than is deserved`.

#### 

