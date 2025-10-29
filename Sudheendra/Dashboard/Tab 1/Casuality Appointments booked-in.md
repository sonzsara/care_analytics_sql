# KPI Query to get the count of Casuality Appointments Booked -in

```sql 

SELECT
    COUNT(public.emr_tokenbooking.id) AS booked_appointments
FROM public.emr_tokenbooking
JOIN public.emr_tokenslot ON public.emr_tokenbooking.token_slot_id = public.emr_tokenslot.id
JOIN public.emr_schedulableresource ON public.emr_tokenslot.resource_id = public.emr_schedulableresource.id
JOIN public.emr_healthcareservice ON public.emr_schedulableresource.healthcare_service_id = public.emr_healthcareservice.id
WHERE 1=1
  AND public.emr_tokenbooking.status = 'booked'
  AND public.emr_healthcareservice.name = 'Casualty'
  [[AND {{booking_date}}]] 
;

```




# Drill down to get the list of Patients booked in for Casuality

```sql

SELECT
    
    public.emr_patient.name AS patient_name,
    public.emr_patient.phone_number AS patient_phone,
    public.emr_patient.gender AS patient_gender,
    public.users_user.first_name || ' ' || public.users_user.last_name AS staff_name
FROM public.emr_tokenbooking
JOIN public.emr_tokenslot ON public.emr_tokenbooking.token_slot_id = public.emr_tokenslot.id
JOIN public.emr_schedulableresource ON public.emr_tokenslot.resource_id = public.emr_schedulableresource.id
JOIN public.emr_healthcareservice ON public.emr_schedulableresource.healthcare_service_id = public.emr_healthcareservice.id
JOIN public.emr_patient ON public.emr_tokenbooking.patient_id = public.emr_patient.id
JOIN public.users_user ON public.emr_tokenbooking.created_by_id = public.users_user.id
WHERE 1=1
  AND public.emr_tokenbooking.status = 'booked'
  AND public.emr_healthcareservice.name = 'Casualty'
  [[AND {{booking_date}}]]
  [[AND public.emr_patient.name ILIKE '%' || {{patient_name}} || '%']] 
  [[AND public.users_user.first_name || ' ' || public.users_user.last_name ILIKE '%' || {{staff_name}} || '%']]  
GROUP BY 
    public.emr_patient.name,
    public.emr_patient.phone_number,
    public.emr_patient.gender,
    public.users_user.first_name,
    public.users_user.last_name
ORDER BY public.emr_patient.name;

```