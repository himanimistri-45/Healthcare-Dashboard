# Healthcare-Dashboard
Healthcare dataset containing patient, hospital, doctor, admission, billing, and report details. Ideal for SQL practice, analysis, ranking, and visualization to explore hospital performance, doctor outcomes, and patient insights.

Dashboard URL:- https://docs.google.com/spreadsheets/d/1o5Dm4brnevIkumZO8poPcENvrlvACE7pzexVk6dOiEk/edit?gid=1898598604#gid=1898598604


Step 1: data structure understand
Columns:
  Blood_Type

  Medical_Condition

  Date_of_Admission

 Doctor

 Hospital

 Insurance_Provider

 Billing_Amount

 Room_Number

 Admission_Type

 Discharge_Date

 Medication

 Test_Results



Step 2: Basic Data 
SELECT * 
FROM Healthcare
LIMIT 10;
👉 some rows see bcz data structure understand



step:3 Would you like me to also give you how to check missing/null values

SELECT 
    SUM(CASE WHEN `Blood Type` IS NULL THEN 1 ELSE 0 END) AS Missing_BloodType,
    SUM(CASE WHEN `Medical Condition` IS NULL THEN 1 ELSE 0 END) AS Missing_Condition,
    SUM(CASE WHEN `Billing Amount` IS NULL THEN 1 ELSE 0 END) AS Missing_Billing,
    SUM(CASE WHEN `Test Results` IS NULL THEN 1 ELSE 0 END) AS Missing_TestResults
FROM healthcare_dataset_1000;
👉 Identify columns with missing values




Step 4: Summary Statistics (Billing Amount)

SELECT 
    MIN(`Billing Amount`) AS Min_Billing,
    MAX(`Billing Amount`) AS Max_Billing,
    AVG(`Billing Amount`) AS Avg_Billing,
    SUM(`Billing Amount`) AS Total_Billing
FROM healthcare_dataset_1000;
👉 Summary of Total Billing

Step 5: Number of Patients by Medical Condition

SELECT 
    `Medical Condition`, 
    COUNT(*) AS Total_Patients
FROM healthcare_dataset_1000
GROUP BY `Medical Condition`;
👉Patients count per medical condition

Step 6: Patients by Admission Type

SELECT 
    `Admission Type`, 
    COUNT(*) AS Patients_Count
FROM healthcare_dataset_1000
GROUP BY `Admission Type`;
👉 Patient count by Admission Type (Emergency, Urgent, Elective)


Step 7: Medication Analysis

SELECT 
    Medication, 
    COUNT(*) AS Usage_Count
FROM healthcare_dataset_1000
GROUP BY Medication
ORDER BY Usage_Count DESC;


Step 8: Test Results Analysis

SELECT 
    `Test Results`, 
    COUNT(*) AS Result_Count
FROM healthcare_dataset_1000
GROUP BY `Test Results`
LIMIT 0, 25;

👉 Patient count by Test Result (Positive, Inconclusive)

Step 9: Doctor wise 

SELECT 
    Doctor, 
    COUNT(*) AS Total_Patients
FROM healthcare_dataset_1000
GROUP BY Doctor
ORDER BY Total_Patients DESC;

👉 Patient count by Doctor

Step 10: Hospital-wise Billing

SELECT 
    Hospital, 
    SUM(`Billing Amount`) AS Total_Hospital_Billing,
    AVG(`Billing Amount`) AS Avg_Hospital_Billing
FROM healthcare_dataset_1000
GROUP BY Hospital
ORDER BY Total_Hospital_Billing DESC
LIMIT 0, 25;
DESCRIBE healthcare_dataset_1000;

👉 Hospital-wise maximum billing

********************************************ADVANCE SQL QUERIES**********************************
Part C:Advanced SQL Queries and Analysis
Objective: Dive deeper into the data using more advanced SQL techniques.
joins,subqueries,ranking

STEP:1 
CREATE OR REPLACE VIEW healthcare_clean AS
SELECT
  `Name`  AS name,
  `Age`   AS age,
  `Gender` AS gender,
  `Blood Type` AS blood_type,
  `Medical Condition` AS medical_condition,
  STR_TO_DATE(`Date of Admission`,'%d-%m-%Y') AS date_of_admission,
  `Doctor` AS doctor,
  `Hospital` AS hospital,
  `Insurance Provider` AS insurance_provider,
  `Billing Amount` AS billing_amount,
  `Room Number` AS room_number,
  `Admission Type` AS admission_type,
  STR_TO_DATE(`Discharge Date`,'%d-%m-%Y') AS discharge_date,
  `Medication` AS medication,
  `Test Results` AS test_results
FROM healthcare;

=>>>>Create a clean view to tackle columns with spaces (OPTIONAL)

STEP:2 Joins (example)
2.1) Aggregate Subquery join (Hospital level averages)
The purpose: To attach each patient’s row with their hospital’s average billing.

WITH hosp AS (
  SELECT hospital, COUNT(*) AS patients, AVG(billing_amount) AS avg_billing
  FROM healthcare_dataset_clean
  GROUP BY hospital
)
SELECT h.name, h.hospital, h.billing_amount, hosp.patients, hosp.avg_billing
FROM healthcare_dataset_clean h
JOIN hosp ON h.hospital = hosp.hospital
ORDER BY h.hospital, h.billing_amount DESC;



2.2) Self-Join (Patients who share the same doctor and the same medical condition (i.e., patient pairs with identical doctor & condition).)
SELECT a.name AS patient1, b.name AS patient2, a.doctor, a.medical_condition
FROM healthcare_dataset_clean a
JOIN healthcare_dataset_clean b
  ON a.doctor = b.doctor
 AND a.medical_condition = b.medical_condition
 AND a.name < b.name;   


2.3) Dimension-like join (Assign an urgency score to each admission type)
SELECT h.name, h.admission_type, s.urgency_score
FROM healthcare_dataset_clean h
JOIN (
  SELECT 'Emergency' AS admission_type, 3 AS urgency_score
  UNION ALL SELECT 'Urgent', 2
  UNION ALL SELECT 'Elective', 1
) AS s USING (admission_type)
ORDER BY urgency_score DESC;

3) Subqueries
Correlated Subquery:- Bills that are higher than the hospital’s average billing.)
SELECT name, hospital, billing_amount
FROM healthcare_dataset_clean h
WHERE billing_amount > (
  SELECT AVG(billing_amount)
  FROM healthcare_dataset_clean
  WHERE hospital = h.hospital
)
ORDER BY hospital, billing_amount DESC;

3.2) IN / EXISTS (All patients of doctors who have ≥2 Positive reports.)
SELECT *
FROM healthcare_dataset_clean
WHERE doctor IN (
  SELECT doctor
  FROM healthcare_dataset_clean
  WHERE test_results = 'Positive'
  GROUP BY doctor
  HAVING COUNT(*) >= 2
);


3.3) Doctor-wise earliest admission (The first admission of each doctor.)
SELECT *
FROM healthcare_dataset_clean h
WHERE date_of_admission = (
  SELECT MIN(date_of_admission)
  FROM healthcare_dataset_clean
  WHERE doctor = h.doctor
);


4) Ranking (Window Functions)
4.1) Rank patients within each hospital based on billing. (Top-N per hospital)
WITH ranked AS (
  SELECT 
    name, hospital, billing_amount,
    DENSE_RANK() OVER (PARTITION BY hospital ORDER BY billing_amount DESC) AS bill_rank
  FROM healthcare_dataset_clean
)
SELECT *
FROM ranked
WHERE bill_rank <= 3
ORDER BY hospital, bill_rank, billing_amount DESC;


4.2) Top Doctors by Total Billing (overall rank)
WITH doc AS (
  SELECT doctor, SUM(billing_amount) AS total_billing, COUNT(*) AS patients
  FROM healthcare_dataset_clean
  GROUP BY doctor
)
SELECT 
  doctor, total_billing, patients,
  RANK() OVER (ORDER BY total_billing DESC) AS rnk
FROM doc
ORDER BY rnk;

4.3) Length-of-Stay (LOS) & Rank (hospital wise)
SELECT
  name, hospital,
  DATEDIFF(discharge_date, date_of_admission) AS los_days,
  RANK() OVER (
    PARTITION BY hospital 
    ORDER BY DATEDIFF(discharge_date, date_of_admission) DESC
  ) AS los_rank
FROM healthcare_dataset_clean
ORDER BY hospital, los_rank;

5) Trend/CTEs (bonus)
5.1) Monthly admissions & billing by condition
SELECT 
  DATE_FORMAT(date_of_admission, '%Y-%m') AS ym,
  medical_condition,
  COUNT(*) AS admits,
  SUM(billing_amount) AS total_bill
FROM healthcare_dataset_clean
GROUP BY ym, medical_condition
ORDER BY ym, medical_condition;



5.2) Running total (month-wise)
WITH m AS (
  SELECT DATE_FORMAT(date_of_admission, '%Y-%m') AS ym,
         SUM(billing_amount) AS amt
  FROM healthcare_dataset_clean
  GROUP BY ym
)
SELECT 
  ym, amt,
  SUM(amt) OVER (ORDER BY ym 
                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM m;


6) Data Quality checks (qucikly)
-- Age outliers
SELECT * FROM healthcare_clean WHERE age < 0 OR age > 120;

-- Missing dates
SELECT * FROM healthcare WHERE `Discharge Date` IS NULL OR `Discharge Date` = '';

-- Invalid negative billing
SELECT * FROM healthcare_clean WHERE billing_amount < 0;













