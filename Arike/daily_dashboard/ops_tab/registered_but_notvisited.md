# KPI Query: Registered But Not Visited Patient Count

```sql

WITH patient_dates AS (
    SELECT DISTINCT 
        DATE(created_date) AS patient_date
    FROM emr_patient
    WHERE deleted = FALSE
      [[AND {{created_date}}]]
),

expanded_booking_dates AS (
    SELECT patient_date FROM patient_dates
    UNION
    SELECT patient_date + INTERVAL '1 day' FROM patient_dates
),

shifted_patients AS (
    SELECT
        emr_patient.id AS patient_id,
        emr_patient.created_date AS pdate,
        emr_patient.instance_tags,
        CASE
            WHEN emr_patient.created_date::time >= '08:00:00'
                 AND emr_patient.created_date::time < '18:00:00' THEN 'day'
            ELSE 'night'
        END AS shift,
        CASE
            WHEN emr_patient.created_date::time >= '00:00:00'
                 AND emr_patient.created_date::time < '08:00:00'
                THEN (emr_patient.created_date::date - INTERVAL '1 day')
            ELSE emr_patient.created_date::date
        END AS shift_date
    FROM emr_patient
    WHERE emr_patient.deleted = FALSE
      AND DATE(emr_patient.created_date) IN (SELECT patient_date FROM expanded_booking_dates)
      
),

final_patients AS (
    SELECT s.*
    FROM shifted_patients s
    WHERE s.shift_date IN (SELECT patient_date FROM patient_dates)
),

home_encounters AS (
    SELECT
        emr_encounter.created_date,
        emr_patient.id AS patient_id
    FROM emr_encounter
    JOIN emr_patient ON emr_encounter.patient_id = emr_patient.id
    WHERE emr_encounter.encounter_class = 'hh'
      AND DATE(emr_encounter.created_date) IN (SELECT patient_date FROM expanded_booking_dates)
),

patient_tags AS (
    SELECT
        fp.patient_id,
        MAX(CASE WHEN et.parent_id = 55 THEN et.display END) AS zone,
        MAX(CASE WHEN et.parent_id = 18 THEN et.display END) AS frequency_of_visit,
        MAX(CASE WHEN et.parent_id = 26 THEN et.display END) AS cross_subsided_model
    FROM final_patients fp
    LEFT JOIN LATERAL unnest(fp.instance_tags) AS tag_id ON TRUE
    LEFT JOIN emr_tagconfig et
        ON et.id = tag_id
       AND et.deleted = FALSE
       AND et.parent_id IN (55, 18, 26)
    GROUP BY fp.patient_id
)

SELECT 
    COUNT(*) AS count
FROM final_patients fp
LEFT JOIN patient_tags pt ON pt.patient_id = fp.patient_id
WHERE fp.shift_date IN (SELECT patient_date FROM expanded_booking_dates)
  AND fp.patient_id NOT IN (SELECT patient_id FROM home_encounters)

  [[AND (pt.zone) ILIKE ({{zone_filter}})]]
  [[AND (pt.frequency_of_visit) ILIKE ({{frequency_filter}})]]
  [[AND (pt.cross_subsided_model) ILIKE ({{cross_subsided_model_filter}})]];

```


# Drill down Query: Registered But Not Visited Patient Details

```sql

WITH patient_dates AS (
    SELECT DISTINCT 
        DATE(created_date) AS patient_date
    FROM emr_patient
    WHERE deleted = FALSE
      [[AND {{created_date}}]]
),

expanded_booking_dates AS (
    SELECT patient_date FROM patient_dates
    UNION
    SELECT patient_date + INTERVAL '1 day' FROM patient_dates
),

shifted_patients AS (
    SELECT
        emr_patient.id AS patient_id,
        emr_patient.name,
        emr_patient.phone_number,
        emr_patient.year_of_birth,
        emr_patient.gender,
        (emr_patient.meta -> 'identifiers' -> 0 ->> 'value') AS adm,
        emr_patient.created_date,
        CASE
            WHEN emr_patient.created_date::time >= '08:00:00'
                 AND emr_patient.created_date::time < '18:00:00' THEN 'day'
            ELSE 'night'
        END AS shift,
        CASE
            WHEN emr_patient.created_date::time >= '00:00:00'
                 AND emr_patient.created_date::time < '08:00:00'
                THEN (emr_patient.created_date::date - INTERVAL '1 day')
            ELSE emr_patient.created_date::date
        END AS shift_date,
        emr_patient.instance_tags
    FROM emr_patient
    WHERE emr_patient.deleted = FALSE
      AND DATE(emr_patient.created_date) IN (SELECT patient_date FROM expanded_booking_dates)
      [[AND {{patient_name}}]]
),

final_patients AS (
    SELECT s.*
    FROM shifted_patients s
    WHERE s.shift_date IN (SELECT patient_date FROM patient_dates)
),

patient_tags AS (
    SELECT
        sp.patient_id,

        MAX(CASE WHEN et.parent_id = 55 THEN et.display END) AS zone,
        MAX(CASE WHEN et.parent_id = 47 THEN et.display END) AS remarks,
        MAX(CASE WHEN et.parent_id = 18 THEN et.display END) AS frequency_of_visit,
        MAX(CASE WHEN et.parent_id = 22 THEN et.display END) AS special_visit,
        MAX(CASE WHEN et.parent_id = 26 THEN et.display END) AS cross_subsided_model

    FROM final_patients sp
    LEFT JOIN LATERAL unnest(sp.instance_tags) AS tag_id ON TRUE
    LEFT JOIN emr_tagconfig et
        ON et.id = tag_id
       AND et.deleted = FALSE
       AND et.parent_id IN (1,10,14,55,30,18,22,26,35,47)
    GROUP BY sp.patient_id
),

home_encounters AS (
    SELECT
        emr_encounter.id AS encounter_id,
        emr_encounter.created_date,
        emr_patient.id AS patient_id
    FROM emr_encounter
    JOIN emr_patient ON emr_encounter.patient_id = emr_patient.id
    WHERE emr_encounter.encounter_class = 'hh'
      AND DATE(emr_encounter.created_date) IN (SELECT patient_date FROM expanded_booking_dates)
)

SELECT 
    final_patients.name,
    final_patients.phone_number,
    final_patients.year_of_birth,
    final_patients.gender,
    final_patients.adm,
    final_patients.shift,
    final_patients.created_date,
    patient_tags.zone,
    patient_tags.remarks,
    patient_tags.frequency_of_visit,
    patient_tags.special_visit,
    patient_tags.cross_subsided_model
FROM final_patients
LEFT JOIN patient_tags 
  ON patient_tags.patient_id = final_patients.patient_id
WHERE final_patients.shift_date IN (SELECT patient_date FROM expanded_booking_dates)
  AND final_patients.patient_id NOT IN (SELECT patient_id FROM home_encounters)

  [[AND (patient_tags.zone) ILIKE ({{zone_filter}})]]
  [[AND (patient_tags.remarks) ILIKE ({{remarks_filter}})]]
  [[AND (patient_tags.frequency_of_visit) ILIKE ({{frequency_filter}})]]
  [[AND (patient_tags.special_visit) ILIKE ({{special_visit_filter}})]]
  [[AND (patient_tags.cross_subsided_model) ILIKE CONCAT('%', ({{cross_subsided_model_filter}}), '%')]]

ORDER BY final_patients.shift_date, final_patients.name ASC, final_patients.phone_number ASC
LIMIT 1048575;

```