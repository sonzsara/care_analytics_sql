# KPI Query: Count of patients registered but no care given

```sql

WITH base_data AS (
    SELECT
        ep.id AS patient_id,
        ep.instance_tags
    FROM public.emr_patient ep
    LEFT JOIN public.emr_encounter ee 
        ON ep.id = ee.patient_id
    WHERE
        (
            ee.status <> 'entered_in_error'
            OR ee.status IS NULL
        )
        AND ee.id IS NULL
        AND (
            ee.encounter_class <> 'vr'
            OR ee.encounter_class IS NULL
        )
),

patient_tags AS (
    SELECT
        bd.patient_id,
        MAX(CASE WHEN et.parent_id = 55 THEN et.display END) AS zone,
        MAX(CASE WHEN et.parent_id = 18 THEN et.display END) AS frequency_of_visit,
        MAX(CASE WHEN et.parent_id = 26 THEN et.display END) AS cross_subsided_model
    FROM base_data bd
    LEFT JOIN LATERAL unnest(bd.instance_tags) AS tag_id ON TRUE
    LEFT JOIN emr_tagconfig et
        ON et.id = tag_id
       AND et.deleted = FALSE
       AND et.parent_id IN (55, 18, 26)
    GROUP BY bd.patient_id
)

SELECT
    COUNT(*) AS "count"
FROM base_data bd
LEFT JOIN patient_tags pt ON pt.patient_id = bd.patient_id
WHERE
    1=1
    [[AND (pt.zone) ILIKE ({{zone_filter}})]]
    [[AND (pt.frequency_of_visit) ILIKE ({{frequency_filter}})]]
    [[AND (pt.cross_subsided_model) ILIKE ({{cross_subsided_model_filter}})]]
;

```


# Drill down query: List of patients registered but no care given

```sql 

WITH base_data AS (
    SELECT
        ep.id AS patient_id,
        ep.name AS patient_name,
        ep.gender,
        ep.phone_number,
        ep.year_of_birth,
        (ep.meta -> 'identifiers' -> 0 ->> 'value') AS adm,
		ep.created_date,
        ep.instance_tags
    FROM public.emr_patient ep
    LEFT JOIN public.emr_encounter ee 
        ON ep.id = ee.patient_id
    WHERE
        (
            ee.status <> 'entered_in_error'
            OR ee.status IS NULL
        )
        AND ee.id IS NULL  
        AND (
            ee.encounter_class <> 'vr'
            OR ee.encounter_class IS NULL
        )
        AND ep.deleted = FALSE
),

patient_tags AS (
    SELECT
        bd.patient_id,
        MAX(CASE WHEN et.parent_id = 55 THEN et.display END) AS zone,
        MAX(CASE WHEN et.parent_id = 47 THEN et.display END) AS remarks,
        MAX(CASE WHEN et.parent_id = 18 THEN et.display END) AS frequency_of_visit,
        MAX(CASE WHEN et.parent_id = 22 THEN et.display END) AS special_visit,
        MAX(CASE WHEN et.parent_id = 26 THEN et.display END) AS cross_subsided_model
    FROM base_data bd
    LEFT JOIN LATERAL unnest(bd.instance_tags) AS tag_id ON TRUE
    LEFT JOIN emr_tagconfig et
        ON et.id = tag_id
       AND et.deleted = FALSE
       AND et.parent_id IN (55, 47, 18, 22, 26)
    GROUP BY bd.patient_id
)

SELECT 
    bd.patient_name,
    bd.gender,
    bd.phone_number,
    bd.year_of_birth,
	bd.created_date,
    bd.adm,
    pt.zone,
    pt.remarks,
    pt.frequency_of_visit,
    pt.special_visit,
    pt.cross_subsided_model
FROM base_data bd
LEFT JOIN patient_tags pt ON pt.patient_id = bd.patient_id


WHERE 1=1
  [[AND (pt.zone) ILIKE ({{zone_filter}})]]
  [[AND (pt.remarks) ILIKE ({{remarks_filter}})]]
  [[AND (pt.frequency_of_visit) ILIKE ({{frequency_filter}})]]
  [[AND (pt.special_visit) ILIKE ({{special_visit_filter}})]]
  [[AND (pt.cross_subsided_model) ILIKE ({{cross_subsided_model_filter}})]]

ORDER BY bd.created_date desc;

```