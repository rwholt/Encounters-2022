# Encounters Analysis 2022

## Table of Contents

- [Project Overview](#project-overview)
- [Data Source](#data-source)
- [Tools](#tools)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Data Analysis](#data-analysis)
- [Results and Findings](#results-and-findings)
- [Recommendations](#recommendations)
### Project Overview
---
This data analysis project aims to provide insights into the hospital patient encounters for the year 2022. By analyzing various aspects of the hospital encounters data, week to identify trends, make data-driven reccomendations, and gain a deeper understanding of the hospital performance.
![image](https://github.com/rwholt/Encounters-2022/assets/130107486/07277f6f-9755-453b-94a5-f613fa4b0b05)

### Data Source
---
Hospital Data: The primary dataset used for this analysis is the Synthea patient records, which contains over 6 million fake patients records.

### Tools
---
- PostgreSQL -Data Analysis
- Tableau - Creating report


### Exploratory Data Analysis
---
 - How much are patients costing the hospital durning an encounter?
 - Has the patient had their flu shots?
 - Has the patient had their covid shots?
 - What is the average length of stay?
 - What is the overall encounters and encounter type?
 - How are patients paying for their care?
 - What are the age groups?
 - What are the most common reasons why patients visit?

### Data Analysis
---
```sql
/*
Objectives
Request 2022 data: 

Overall numbers of encounter types 

Pay distribution for the different payor types 

% of patients who had full covid immunity (2 shots) BEFORE the encounter 

% of patients who had full flu immunity (1 seasonal shot) BEFORE the encounter 

Base cost for the hospital per encounter type 

Median Time of the encounter 

Average LOS 

Age distribution 

Most Common Encounter Reasons 
*/ 


with icd_crosswalk as  				--CTE that shows Overall numbers of encounter types 

(
  select distinct description,
         code
  from postgres.public.conditions
),

flu as
(
  select patient,
	     min(date) as date
  from postgres.public.immunizations
  where date between '2022-01-01 00:00' and '2023-01-01 00:00'
    and code = '5302' 				-- This is the seasonal flu vaccine ICD
  group by patient
),

covid as
(
  select patient,
	     date,
	     row_number() over (partition by patient order by date asc) as seq
  from postgres.public.immunizations
  where lower(description) like '%covid%' 	-- this wildcard searches everything covid
),

covid_final as
(
	select patient,
	       date
	from covid
	where seq = 2 
)

select enc.id as encounter_id,
       enc.encounterclass,
	   enc.description as enc_type,
	   enc.base_encounter_cost,
	   enc.start,
	   enc.stop,
	   case when flu.date < enc.start then 1
	        else 0
			end as flu_2022,
	   case when cov.date < enc.start then 1
	        else 0
			end as covid,
	   icd.description as enc_reason,
	   pay.name as payer,
	   pay.ownership as payer_category,
	   pat.birthdate,
	   pat.first,
	   pat.last,
	   pat.zip,
	   pat.race,
	   pat.ethnicity,
	   pat.id as patient_id
from postgres.public.encounters as enc
left join postgres.public.payers as pay		--Join grants access to determine pay distribution for the different payor types 
  on enc.payer = pay.id
left join postgres.public.patients as pat	--Join grants access to determine age distribution 
  on enc.patient = pat.id
join icd_crosswalk as icd			--Join grants access to CTE name icd to determine icd code and description
  on enc.reasoncode = icd.code
left join flu					--Join grants access to determine % of patients who had full flu immunity (1 seasonal shot) BEFORE the encounter 
  on enc.patient = flu.patient
left join covid_final as cov			--Join grants access to determine % of patients who had full covid immunity (2 shots) BEFORE the encounter 
  on enc.patient = cov.patient
where enc.stop >= '2022-01-01 00:00'
  and enc.stop <  '2023-01-01 00:00'
```

### Results and Findings
---
The analysis results are summarized as follows:
1. Outpatient encounters cost the hospital the most money.
2. Over 87% of patients admitted into the hospital have recieve flu and both covid shots.
3. The average length of stay for patients that go to the ER is 1 hr.
4. Outpatient receives over 3k+ patients.
5. The top reason is "Patient with a problem" from the Outpatient department.


### Recommendations
---

Based on the analysis, we reccomend the following actions
- Increase staff to meet the patient demand in the summer time for the ER.

