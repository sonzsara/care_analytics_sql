# KPI: Count of New Patients Registered

```sql 

WITH patient_dates AS (
    SELECT DISTINCT 
        DATE(created_date) AS patient_date
    FROM emr_patient
    WHERE deleted = FALSE
      [[AND {{created_date}}]]
),


extended_patient_dates AS (
    SELECT patient_date
    FROM patient_dates
    UNION ALL
    SELECT MAX(patient_date) + INTERVAL '1 day'
    FROM patient_dates
),

shifted_patients AS (
    SELECT
        id AS patient_id,
        created_date,
        instance_tags,
        CASE
            WHEN created_date::time >= '08:00:00'
                 AND created_date::time < '20:00:00' THEN 'day'
            ELSE 'night'
        END AS shift,
        CASE
            WHEN created_date::time >= '00:00:00'
                 AND created_date::time < '08:00:00'
                THEN (created_date::date - INTERVAL '1 day')
            ELSE created_date::date
        END AS shift_date
    FROM emr_patient
    WHERE deleted = FALSE
      AND DATE(created_date) IN (SELECT patient_date FROM extended_patient_dates)
      AND NOT (
          DATE(created_date) = (SELECT MAX(patient_date) FROM extended_patient_dates)
          AND created_date::time > '08:00:00'
      )
),

patient_tags AS (
    SELECT
        sp.patient_id,
        MAX(CASE WHEN et.parent_id = 55 THEN et.display END) AS zone,
        MAX(CASE WHEN et.parent_id = 18 THEN et.display END) AS frequency_of_visit,
        MAX(CASE WHEN et.parent_id = 26 THEN et.display END) AS cross_subsided_model
    FROM shifted_patients sp
    LEFT JOIN LATERAL unnest(sp.instance_tags) AS tag_id ON TRUE
    LEFT JOIN emr_tagconfig et
        ON et.id = tag_id
       AND et.deleted = FALSE
       AND et.parent_id IN (55, 18, 26)
    GROUP BY sp.patient_id
)

SELECT COUNT(*) AS count
FROM shifted_patients sp
LEFT JOIN patient_tags pt ON pt.patient_id = sp.patient_id
WHERE sp.shift_date IN (SELECT patient_date FROM extended_patient_dates)
  [[AND (pt.zone) ILIKE ({{zone_filter}})]]
  [[AND (pt.frequency_of_visit) ILIKE ({{frequency_filter}})]]
  [[AND (pt.cross_subsided_model) ILIKE ({{cross_subsided_model_filter}})]];


```


#  Drill down: List of New Patients Registered

```sql

WITH patient_dates AS (
    SELECT DISTINCT 
        DATE(created_date) AS patient_date
    FROM emr_patient
    WHERE deleted = FALSE
      [[AND {{created_date}}]]
),

extended_patient_dates AS (
    SELECT patient_date
    FROM patient_dates
    UNION ALL
    SELECT MAX(patient_date) + INTERVAL '1 day'
    FROM patient_dates
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
      AND DATE(emr_patient.created_date) IN (SELECT patient_date FROM extended_patient_dates)
      AND NOT (
          DATE(emr_patient.created_date) = (SELECT MAX(patient_date) FROM extended_patient_dates)
          AND emr_patient.created_date::time > '08:00:00'
      )
      [[AND {{patient_name}}]]
),

patient_tags AS (
    SELECT
        sp.patient_id,
        STRING_AGG(CASE WHEN et.parent_id = 55 THEN et.display END, ', ') AS zone,
        STRING_AGG(CASE WHEN et.parent_id = 47 THEN et.display END, ', ') AS remarks,
        STRING_AGG(CASE WHEN et.parent_id = 18 THEN et.display END, ', ') AS frequency_of_care,
        STRING_AGG(CASE WHEN et.parent_id = 22 THEN et.display END, ', ') AS special_note,
        STRING_AGG(CASE WHEN et.parent_id = 26 THEN et.display END, ', ') AS cross_subsided_model
    FROM shifted_patients sp
    LEFT JOIN LATERAL unnest(sp.instance_tags) AS tag_id ON TRUE
    LEFT JOIN emr_tagconfig et 
        ON et.id = tag_id AND et.deleted = FALSE
    GROUP BY sp.patient_id
)

SELECT 
    sp.name,
    sp.phone_number,
    sp.year_of_birth,
    sp.gender,
    sp.adm,
    sp.shift,
    sp.created_date,
    pt.zone,
    pt.remarks,
    pt.frequency_of_care,
    pt.special_note,
    pt.cross_subsided_model
FROM shifted_patients sp
LEFT JOIN patient_tags pt 
  ON pt.patient_id = sp.patient_id
WHERE sp.shift_date IN (SELECT patient_date FROM extended_patient_dates)

  [[AND (pt.zone) ILIKE ({{zone_filter}})]]
  [[AND (pt.remarks) ILIKE ({{remarks_filter}})]]
  [[AND (pt.frequency_of_care) ILIKE ({{frequency_filter}})]]
  [[AND (pt.special_note) ILIKE ({{special_note_filter}})]]
  [[AND (pt.cross_subsided_model) ILIKE CONCAT('%', ({{cross_subsided_model_filter}}), '%') ]]

ORDER BY sp.shift_date, sp.name ASC, sp.phone_number ASC
LIMIT 1048575;

```