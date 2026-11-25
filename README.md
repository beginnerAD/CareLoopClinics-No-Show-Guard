
CareLoop Clinics — No-Show Guard
Repository: https://github.com/beginnerAD/CareLoop-Clinics-No-Show-Guard

Summary of Findings
Part 1 (Data Cleaning): Cleaned 1,020 raw records down to 990 usable appointments (218 no-shows, 22.0% overall) in appointments_clean.csv after removing 20 duplicates, dropping 10 rows with invalid dates or blank statuses, imputing missing ages/distances, and capping extreme fee/distance outliers.
Part 2 (SQL Analysis): Queries across the 990 cleaned visits showed that bookings made 31+ days ahead missed at a 60.5% rate (vs. 16.2% for 1–7 days), and Gomti Nagar Care Point (Lucknow) recorded the highest clinic no-show rate at 33.3% (30 of 90 visits).
Part 3 (No-Show Prediction): Picked Logistic Regression (test ROC-AUC 0.7889, 5-fold CV ROC-AUC 0.7737, cutoff threshold 0.35) over the baseline and tree models because it raised test recall to 57.1% (catching 28 of 49 test no-shows) while keeping precision at 57.1%.
Part 4 (Call Note Triage): Ran 10 follow-up notes through Gemini with Pydantic validation (14 API calls total); 5 of the 10 notes had no reminder sent, and both patients who said they forgot (N001 and N007) were in that unreminded group.
Recommendation
CareLoop Clinics should enforce automated reminders across all bookings—especially visits booked more than 14 days in advance (where no-show rates jump from 16.2% to 30.0%–60.5%)—and have the front desk make evening confirmation calls to patients flagged above our 0.35 probability threshold by the Logistic Regression model (test ROC-AUC 0.7889). Across our 990 cleaned appointments (22.0% overall no-show rate) and the 10 call-centre notes, half (5 of 10) of sampled no-shows never received a reminder, and 100% (2 of 2) of patients who simply forgot were unreminded. Focusing daily outreach on the ~25% of appointments flagged above 0.35 gives call-centre staff a manageable call list that directly targets preventable no-shows without wasting outreach capacity.

Limitations
Small note sample: Ten follow-up notes are only enough to validate the classification pipeline and cannot represent the true breakdown of no-show reasons across all eight clinics.
Data leakage in the split: Imputation medians and outlier fences were calculated across all 990 rows before splitting, and because the split was random rather than chronological, the same patient can appear in both the training and test sets.
Observational records and fixed cutoff: The historical data shows associations rather than proof of cause and effect, and the optimal risk cutoff in live operations would depend on how many evening calls the front desk can realistically make against the cost of an empty doctor slot.
1. Executive Summary & Key Metrics
Pipeline Stage	Metric / Deliverable	Value
Data Ingestion (01_ingest_clean.ipynb)	Raw Records Parsed (appointments.csv)	1,020 rows
Data Cleaning (01_ingest_clean.ipynb)	Validated Appointments (appointments_clean.csv)	990 rows
Data Quarantine (01_ingest_clean.ipynb)	Rejected Malformed Rows (rejected_rows.json)	30 rows
Target Distribution	Overall Appointment No-Show Rate	22.0% (218 / 990)
Weather Enrichment (clinics_enriched.csv)	Clinic Locations Enriched via Open-Meteo API	8 clinics (temp_c)
SQL Seeding (load_data.sql)	Total INSERT Statements Generated	998 rows (8 clinics + 990 appointments)
Predictive Modeling (03_model.ipynb)	Majority-Class Baseline Accuracy	75.25%
Best Classifier (03_model.ipynb)	Logistic Regression (StandardScaler Pipeline)	0.7889 Test ROC-AUC (0.7737 5-Fold CV)
GenAI Call-Note Triage (04_genai.ipynb)	Structured Notes Analysed (notes_analysed.csv)	10 notes (14 total API calls)
2. Part 1 — Data Ingestion, Cleaning & Weather Enrichment (01_ingest_clean.ipynb)
Strict Schema Parsing: Read data/appointments.csv using Python's csv.DictReader without relying on automatic type inference. Cast age, fee_inr, and prior_no_shows to int and distance_km to float.
Validation & Quarantine (rejected_rows.json): Isolated 30 rows failing type conversion, missing mandatory fields, or violating domain rules (invalid dates, negative fees/distances, out-of-range ages) into rejected_rows.json with explicit rejection reasons.
Feature Engineering (appointments_clean.csv):
Standardized date formats (YYYY-MM-DD) and computed lead_days (appt_date - booked_on, ranging from 1 to 45 days with a mean of 11.82 days).
Normalized categorical fields (reminder_sent, booking_channel, visit_type) and encoded the binary target no_show (1 = No-Show, 0 = Attended).
Live Weather API Enrichment (clinics_enriched.csv): Queried the Open-Meteo forecast API using each clinic's latitude and longitude to append current hourly temperature (temp_c) to data/clinics.csv.
Database Seed Generation (load_data.sql): Programmatically generated SQL INSERT statements for all 8 clinics followed by all 990 clean appointments to ensure foreign-key integrity.
3. Part 2 — Relational Database & SQL Analytics (load_data.sql & 02_queries.sql)
Loaded clinics and appointments into SQLite and executed 6 validation and operational queries in 02_queries.sql:

Row-Count & Integrity Verification: Confirmed 8 clinic records and 990 appointment records loaded with zero orphaned foreign keys.
Overall & Clinic-Level No-Show Rates: Ranked clinics by total bookings, missed appointments, no-show percentage, and lost consultation revenue (SUM(fee_inr) where no_show = 1).
Reminder Effectiveness Analysis: Compared attendance rates between reminder_sent = 'Yes' and reminder_sent = 'No', confirming that unreminded appointments fail at a substantially higher rate.
Lead-Time Bucket Breakdown: Grouped bookings by lead_days intervals (1–7, 8–14, 15–30, 31+ days), showing no-show risk rises sharply once lead times exceed two weeks.
Channel & Visit-Type Segmentation: Cross-analyzed booking_channel and visit_type to pinpoint high-friction booking flows.
High-Risk Patient Cohort Identification: Flagged patients with prior_no_shows >= 2 and long travel distances (distance_km) for proactive front-desk outreach.
4. Part 3 — Predictive Modeling & Risk Scoring (03_model.ipynb)
Feature Preparation & Split: Mapped reminder_sent (Yes -> 1, No -> 0), one-hot encoded booking_channel and visit_type via pd.get_dummies(), and split the 990 records into an 80/20 train/test split (792 train, 198 test, random_state=42).
Model Benchmarking:
Model	5-Fold CV ROC-AUC (Train)	Test Accuracy	Test ROC-AUC	Key Takeaway
Majority Baseline (Always 0)	—	75.25%	0.5000	Predicts zero no-shows (0% Recall)
Logistic Regression (Pipeline)	0.7737	78.79%	0.7889	Selected Winner — best generalization & calibration
Decision Tree (Unpruned, Depth 15)	—	67.17%	—	Severe overfitting (1.0000 train vs. 0.6717 test)
Decision Tree (max_depth=3)	0.6784	77.78%	—	Pruning eliminates memorization and lifts test accuracy
Random Forest (100 trees)	0.7466	74.75%	0.7057	Top features: lead_days (0.220), age (0.197), distance_km (0.195)
Random Forest (Tuned GridSearchCV)	0.7673	—	—	Best params: max_depth=3, n_estimators=200
Odds-Ratio Insight: Inspecting Logistic Regression coefficients confirmed that sending a reminder (reminder_sent = 1) substantially reduces the odds of a patient no-show (exp(coef / std) < 1).
Threshold Selection & Export (test_risk_scores.csv): Evaluated thresholds 0.50 vs. 0.35 on out-of-fold training probabilities (cross_val_predict) to maximize F1-score without touching the test set, then scored X_test once and saved appointment_id, no_show, risk_prob, and flag to test_risk_scores.csv.
5. Part 4 — GenAI Call-Note Triage (04_genai.ipynb)
Schema-Enforced Extraction (notes_analysed.csv): Built an extraction pipeline using gemini-flash-lite-latest (temperature=0, response_mime_type="application/json") paired with a strict Pydantic FollowupInsight validator enforcing 6 allowed root_cause labels: forgot_or_no_reminder, distance_or_transport, cost_or_fee, illness_or_emergency, work_or_schedule, and other.
Edge-Case & Resilience Testing:
Handled blank/missing notes (N004) deterministically as "skipped_blank" without wasting API quota (14 total API calls across Tasks 4.1–4.3).
Verified Pydantic ValidationError handling against out-of-range confidence values (1.7) and missing required keys.
Prompt Engineering (V1 Minimal vs. V2 CRAFT): Tested both prompts on Hinglish (N007) and ambiguous (N009) call notes. V2 (CRAFT) outperformed V1 by enforcing closed-set category boundaries and instructing the model to assign "other" on ambiguous notes rather than hallucinating a specific cause.
Cross-Tabulation & Operational Recommendations: Joined notes_analysed.csv with appointments_clean.csv on appointment_id and cross-tabulated root_cause against reminder_sent:
Automate Dual-Stage Reminders: Trigger automated SMS/WhatsApp reminders 24 hours and 2 hours prior to visits for all bookings with lead_days > 14 where reminder_sent = 'No' to eliminate forgot_or_no_reminder misses.
Targeted Tele-Consult & Rescheduling Outreach: Use test_risk_scores.csv flags to proactively offer tele-consultations or flexible rescheduling to patients with long distance_km (distance_or_transport) or repeat prior_no_shows.
6. Repository Structure
CareLoop-Clinics-No-Show-Guard/
├── README.md                  # Project documentation, methodology, metrics, and findings
├── 01_ingest_clean.ipynb      # Part 1: Raw CSV validation, cleaning, Open-Meteo API enrichment, SQL generation
├── 02_queries.sql             # Part 2: 6 SQLite validation and operational analytical queries
├── 03_model.ipynb             # Part 3: Feature encoding, model benchmarking, tuning, and risk scoring
├── 04_genai.ipynb             # Part 4: Gemini structured extraction, Pydantic validation, V1/V2 prompt comparison
├── appointments_clean.csv     # Cleaned dataset (990 validated rows)
├── clinics_enriched.csv       # Clinic metadata enriched with Open-Meteo temperature (8 rows)
├── rejected_rows.json         # Quarantined malformed rows with rejection reasons (30 rows)
├── load_data.sql              # SQLite INSERT statements for clinics and appointments (998 statements)
├── test_risk_scores.csv       # Out-of-sample test predictions, probabilities, and risk flags (198 rows)
├── notes_analysed.csv         # Structured GenAI call-note classifications and validation statuses (10 rows)
└── data/
    ├── appointments.csv       # Raw appointment records (1,020 rows)
    ├── clinics.csv            # Raw clinic directory (8 rows)
    └── followup_notes.csv     # Raw post-appointment call-centre notes
