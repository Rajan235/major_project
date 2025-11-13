Project Report: Telecom Fraud Detection Model (v3)

Meeting Date: November 13, 2025
Prepared For: Project Stakeholders
Prepared By: Rajan (Student, Computer Science)

---

## 1. Executive Summary

This project has successfully developed a high-accuracy, multi-class fraud detection model (dubbed "v3") capable of identifying 5 out of 6 complex fraud types, including the notoriously difficult `SIM_SWAP` attack.

The final "v3" model achieves **99% overall accuracy** and, most critically, improved the **`SIM_SWAP` F1-score from 0.00 to 0.67**.

This breakthrough was achieved by moving from simple _stateless_ analysis (v1/v2) to _stateful, behavioral_ feature engineering (v3). We proved that the key to detecting `SIM_SWAP` is not the event itself, but the _change in state_—specifically, the **`imei_changed`** flag, which we engineered.

The model is ready for a production pipeline, but its _stateful nature_ has specific architectural requirements for deployment (see Section 6).

---

## 2. The Project Journey: Model Evolution

Our development followed an iterative process, with each version providing critical insights.

- **Model v1 (Baseline):**

  - **Method:** Used raw data with basic `LabelEncoder` for categorical features.
  - **Result:** 96% accuracy, but a **total failure on `SIM_SWAP` (0.06 F1-score)**.
  - **Insight:** SHAP analysis showed the model was confusing `SIM_SWAP` with 'Normal' traffic.

- **Model v2 (Stateless Feature Engineering):**

  - **Method:** Engineered new _stateless_ features like `is_night`, `bytes_per_duration`, and `ip_src_rarity`.
  - **Result:** 99% accuracy, but the **`SIM_SWAP` F1-score dropped to 0.00**.
  - **Insight:** Our new "rarity" features were _too good_ at identifying 'Normal' traffic, completely "drowning out" the weak `SIM_SWAP` signal. This proved we needed a new _type_ of feature.

- **Model v3 (Stateful Feature Engineering):**
  - **Method:** Engineered a _stateful, behavioral_ feature: **`imei_changed`**. This required sorting the entire dataset by `imsi` and `timestamp` to find where an `imsi` (SIM card) was paired with a new `imei` (phone).
  - **Result:** **Success.** This single feature provided the "golden signal" to solve our problem.

---

## 3. Key Insights from Data Analysis (EDA & SHAP)

Our analysis of the raw data and model behavior was essential.

1.  **"Golden Features" Found:** We found simple, powerful rules for specific attacks:

    - **`DATA_EXFIL`:** 100% of these attacks used a **`vpn_usage`**.
    - **`MASS_SMS`:** These attacks were perfectly identified by our **`ip_src_rarity`** and **`domain_rarity`** features.

2.  **`SIM_SWAP` Was Statistically Invisible:** Our EDA proved that `SIM_SWAP` events, when viewed in isolation, are identical to 'Normal' traffic (same duration, data usage, etc.). This justified the need for stateful, v3 features.

3.  **Target Leakage Identified & Dropped:** We found the **`is_fraud`** column was a binary "cheat sheet" for our target. It was **dropped** to ensure the model learns real patterns.

4.  **Redundant & Useless Features Dropped:**
    - `location_lon` was dropped (1.00 correlation with `location_lat`).
    - `cell_id_rarity` was dropped (0.00 correlation with the target).

---

## 4. Final "v3" Model Performance

The "v3" model is highly effective and reliable. The addition of the `imei_changed` feature solved our primary challenge.

| Class                | F1-Score (v2) | F1-Score (v3) | Analysis              |
| :------------------- | :-----------: | :-----------: | :-------------------- |
| **DATA_EXFIL**       |     0.98      |   **0.97**    | **Success.**          |
| **MASS_SMS**         |     1.00      |   **1.00**    | **Success.**          |
| **OTP_REDIRECTION**  |     1.00      |   **1.00**    | **Success.**          |
| **VOIP_SPOOF**       |     1.00      |   **1.00**    | **Success.**          |
| **`SIM_SWAP`**       |     0.00      |   **0.67**    | **MAJOR IMPROVEMENT** |
| **Normal**           |     0.99      |   **1.00**    | **Success.**          |
| **Overall Accuracy** |      99%      |    **99%**    |                       |

The "v3" model's **precision** for `SIM_SWAP` is **0.86 (86%)**, meaning when it flags a `SIM_SWAP`, it is correct 86% of the time.

---

## 5. Improvements Needed & Discussion Points (For the Meeting)

Based on these results, here is what we need to discuss.

#### 1. The Production Challenge: A Stateful Kafka Consumer

- **The Issue:** Our `v3` model _requires_ the `imei_changed` feature. A simple Kafka consumer (like we originally planned) cannot create this feature because it has no "memory" of past events.
- **The Solution:** We must design a **stateful streaming architecture**.
- **Proposal:**
  1.  The Kafka consumer script must connect to a **fast key-value database (like Redis)**.
  2.  This database will store the last known state for every `imsi` (e.g., `Key: "imsi_A"`, `Value: "imei_X"`).
  3.  When a new message arrives (`imsi_A`, `imei_Y`), the script will:
      - Fetch the last state (`imei_X`) from Redis.
      - Compare `imei_X` to `imei_Y` to create the `imei_changed` feature (0 or 1).
      - Feed this feature to the `v3` model for a prediction.
      - Update the database with the new state: `SET "imsi_A" "imei_Y"`.

#### 2. Model Improvement: Getting `SIM_SWAP` to 99%

- **The Issue:** Our `imei_changed` feature was a huge success, but it still missed ~30% of `SIM_SWAP` attacks (those 62 events).
- **The Solution:** We need to engineer _more_ stateful, behavioral features to catch the rest.
- **Features to Brainstorm:**
  - `user_id_changed_for_imsi` (Did the `user_id` on this `imsi` suddenly change?)
  - `geo_velocity_anomaly` (Did the `cell_id` jump 1000km in 5 minutes?)
  - `account_age` (Is this `imsi` brand new?)

#### 3. Hyperparameter Tuning

- We have not yet tuned the XGBoost model. We can likely improve the 0.67 F1-score even further by tuning parameters like `max_depth`, `learning_rate`, etc.

#### 4. Explore Advanced Architectures

- Our final, stateful dataset is a perfect candidate for more advanced models like **GNNs (Graph Neural Networks)**, which are designed to analyze relationships (`user` -> `imsi` -> `imei`).
