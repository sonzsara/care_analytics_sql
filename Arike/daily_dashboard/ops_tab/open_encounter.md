# KPI CARD: Count of Open Encounters with Tag Filters
 
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
        ee.status <> 'completed'
        OR ee.status IS NULL
      )
      AND (
        ee.status <> 'entered_in_error'
        OR ee.status IS NULL
      )
      AND ee.facility_id = 2
      AND ep.deleted = FALSE
      AND ee.deleted = FALSE
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
),

encounters_with_tags AS (
    SELECT
        ep.id AS patient_id
    FROM public.emr_patient ep
    LEFT JOIN public.emr_encounter ee 
      ON ep.id = ee.patient_id
    LEFT JOIN patient_tags pt ON pt.patient_id = ep.id
    WHERE
      (
        ee.status <> 'completed'
        OR ee.status IS NULL
      )
      AND (
        ee.status <> 'entered_in_error'
        OR ee.status IS NULL
      )
      AND ee.facility_id = 2
      AND ep.deleted = FALSE
      AND ee.deleted = FALSE
      [[AND LOWER(pt.zone) ILIKE LOWER({{zone_filter}})]]
      [[AND LOWER(pt.frequency_of_visit) ILIKE LOWER({{frequency_filter}})]]
      [[AND LOWER(pt.cross_subsided_model) ILIKE LOWER({{cross_subsided_model_filter}})]]
)

SELECT COUNT(*) AS patient_count
FROM encounters_with_tags;

```

# Drilldown Query: List of Open Encounters with Tag Filters

```sql

WITH base_data AS (
    SELECT
        ep.id AS patient_id,
        ep.name AS patient_name,
        ep.gender,
        ep.phone_number,
        ep.year_of_birth,
        (ep.meta -> 'identifiers' -> 0 ->> 'value') AS adm,
        uu.first_name || ' ' || uu.last_name AS nurse_doctor_name,
        ee.created_date,
        ep.instance_tags
    FROM public.emr_patient ep
    LEFT JOIN public.emr_encounter ee 
      ON ep.id = ee.patient_id
    LEFT JOIN public.users_user uu 
      ON ee.created_by_id = uu.id
    WHERE
      (
        ee.status <> 'completed'
        OR ee.status IS NULL
      )
      AND (
        ee.status <> 'entered_in_error'
        OR ee.status IS NULL
      )
      AND ee.facility_id = 2
      AND ep.deleted = FALSE
      AND ee.deleted = FALSE
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
    bd.adm,
    bd.nurse_doctor_name,
    bd.created_date,
    pt.zone,
    pt.remarks,
    pt.frequency_of_visit,
    pt.special_visit,
    pt.cross_subsided_model
FROM base_data bd
LEFT JOIN patient_tags pt ON pt.patient_id = bd.patient_id

WHERE 1=1
[[AND LOWER(pt.zone) ILIKE LOWER({{zone_filter}})]]
[[AND LOWER(pt.remarks) ILIKE LOWER({{remarks_filter}})]]
[[AND LOWER(pt.frequency_of_visit) ILIKE LOWER({{frequency_filter}})]]
[[AND LOWER(pt.special_visit) ILIKE LOWER({{special_visit_filter}})]]
[[AND LOWER(pt.cross_subsided_model) ILIKE LOWER({{cross_subsided_model_filter}})]]

ORDER BY bd.created_date desc;

```