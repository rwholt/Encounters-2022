# Encounters Analysis 2022

## Table of Contents

- [Project Overview](#project-overview)
- [Data Source](#data-source)
- [Tools](#tools)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Data Analysis](#data-analysis)
- [Results and Findings](#results-and-findings)
- [Recommendations](#recommendations)
### **Project Overview**
---
This data analysis project aims to provide insights into the hospital patient encounters for the year 2022. By analyzing various aspects of the hospital encounters data, week to identify trends, make data-driven reccomendations, and gain a deeper understanding of the hospital performance. Click <a href="https://public.tableau.com/app/profile/reginald.holt/viz/Encounter2022Dashboard/Dashboard3" style="color: #FF5733;">here</a> to view the dashboard.


### **Data Source**
---
Hospital Data: The primary dataset used for this analysis is the Synthea patient records, which contains over 6 million fake patients records.

### **Tools**
---
- PostgreSQL -Data Analysis
- Tableau - Creating report


### **Exploratory Data Analysis**
---
 - How much are patients costing the hospital durning an encounter?
 - Has the patient had their flu shots?
 - Has the patient had their covid shots?
 - What is the average length of stay?
 - What is the overall encounters and encounter type?
 - How are patients paying for their care?
 - What are the age groups?
 - What are the most common reasons why patients visit?

### **Data Analysis**
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

### **Results and Findings**
---
The analysis results are summarized as follows:
1. Outpatient encounters cost the hospital the most money.
2. Over 87% of patients admitted into the hospital have recieve flu and both covid shots.
3. The average length of stay for patients that go to the ER is 1 hr.
4. Outpatient receives over 3k+ patients.
5. The top reason is "Patient with a problem" from the Outpatient department.


### **Recommendations**
---

Based on the analysis, we reccomend the following actions
### ER
- **Narrative for Encounter 2022 Dashboard**

This dashboard presents a comprehensive overview of emergency encounters throughout 2022, tracking key metrics such as average duration, patient count, cost, and payer distribution, as well as COVID-19 immunity status among patients. A notable trend is the fluctuation in average emergency encounter duration, peaking in July and December. This data is pivotal for identifying bottlenecks and improving patient flow. The median emergency time stands at 60 minutes, with 764 unique patients encountered, indicating a substantial demand on emergency services.

The financial aspect is represented in the "Number of Emergency Encounters vs Base Cost" chart, showing a non-linear relationship between encounter frequency and associated costs, suggesting variable costs that may not be solely dependent on the number of encounters. The "Payer Treemap" reveals Medicaid as the predominant payer, covering 23% of encounters, followed by various private insurers. The dashboard also highlights that 68% of emergency encounters involved patients with full COVID immunity, a crucial metric in the ongoing management of the pandemic.

**Actionable Insights**

1. **Emergency Duration Management**: Investigate the cause of peaks in average emergency encounter duration in July and December. Assess staffing levels, patient flow processes, and seasonal illnesses that might contribute to longer wait times. Implement strategies to streamline operations during expected high-demand periods.

2. **Cost Analysis**: Analyze the base cost data to understand the factors driving cost independently of the number of encounters. This could include resource utilization, staffing costs, or the complexity of cases. Identify opportunities for cost-saving measures that do not compromise patient care.

3. **Payer Engagement**: Engage with Medicaid and other major payers identified in the treemap to discuss the management of emergency encounters and explore opportunities for cost-sharing or efficiency programs.

4. **COVID-19 Immunity Tracking**: Continue to monitor the COVID immunity status of patients as it impacts hospital resources and infection control measures. Consider comparing the emergency encounter immunity rate with that of the general population to inform public health strategies.

5. **Types of Emergency Encounters**: Address the high number of Obstetrics Emergency encounters by evaluating obstetrics care processes and patient education programs. Explore whether these emergencies can be reduced through better prenatal care or early intervention strategies.

### **Inpatient**
- **Narrative for Inpatient Encounter 2022 Dashboard**

The Inpatient Encounter 2022 Dashboard presents key metrics that detail the inpatient care landscape over the course of the year. With a median inpatient stay of approximately 3.5 days, the facility served 108 unique patients, indicating a focused yet significant impact on the healthcare system. The payer mix is heavily dominated by Medicare, which covers nearly half of the inpatient encounters, reflecting the patient demographic likely skewed towards older adults or those with certain disabilities. The data also shows an impressive 87% of encounters involved patients with full COVID immunity, suggesting a high rate of vaccination or previous infection among this patient group.

The Average Days in Inpatient Encounter graph indicates a variable length of stay throughout the year, with a notable decrease in September. The graph displaying the number of inpatient encounters versus base cost suggests an upward trend in costs despite fluctuations in encounter numbers, pointing to varying degrees of care complexity or resource usage that may not be directly proportional to patient volume.

**Actionable Insights**

1. **Length of Stay Optimization**: Investigate the reasons behind the fluctuation in the average length of stay, particularly the dip in September. Analyze factors such as discharge policies, care pathways efficiency, and potential for early rehabilitation programs to maintain or reduce the length of stay where appropriate.

2. **Cost Management**: Examine the factors contributing to the increase in base costs, especially in months where the number of encounters is not at its peak. This could involve a deep dive into the types of services provided, staffing models, and supply chain management to identify cost-saving opportunities without compromising care quality.

3. **Payer Strategy Enhancement**: With Medicare being the predominant payer, ensure that billing and coding practices are optimized for this group. Additionally, explore opportunities for engaging with dual-eligible and private payers to improve contract terms, streamline the payment process, and ensure timely reimbursement.

4. **Service Type Analysis**: The significant number of hospital encounters with procedures suggests a highly active procedural unit. Assess the efficiency and outcomes of these encounters to ensure resources are being used optimally. For transplant encounters and other specialized services, consider capacity planning to meet patient needs and maintain high-quality care.

5. **COVID-19 Immunity and Infection Control**: Given the high percentage of fully immune patients, continue to prioritize infection control measures to protect both patients and staff, particularly as new variants emerge. Additionally, maintain robust data on immunity rates as this impacts staffing requirements, isolation room availability, and PPE usage.

### **Outpatient**
-**Narrative for Outpatient Encounter 2022 Dashboard**

The Outpatient Encounter 2022 Dashboard illustrates various metrics regarding outpatient services throughout the year. With over 3,000 unique patients and a median visit duration of 123 minutes, the department seems to handle a substantial outpatient volume with considerable time per visit. The average minutes in outpatient encounters have remained relatively consistent, with minor fluctuations throughout the year, indicating a stable process in managing outpatient visits.

Prenatal care encounters are the most frequent, followed by encounters with procedures and follow-up appointments. This suggests a high engagement with expectant mothers and a strong procedural component in outpatient care. Financially, the number of outpatient encounters correlates with the base cost, showing an increasing trend in costs as the year progresses, which could reflect both the volume of services provided and the complexity or resource intensity of those services.

Medicare is the primary payer, accounting for a third of the encounters, indicating a significant proportion of the outpatient population may be elderly or have disabilities. In addition, 79% of encounters involved patients with full COVID immunity, a metric that has likely played a role in infection control and patient throughput efficiencies.

**Actionable Insights**

1. **Efficiency in Visit Duration**: While the median visit time is over two hours, it's important to assess whether this duration aligns with the complexity of services provided. Consider evaluating workflow processes, appointment scheduling, and staffing levels to optimize patient throughput without compromising the quality of care.

2. **Prenatal Care Focus**: Given the high volume of prenatal care encounters, ensure that this service is adequately resourced and that best practices are followed. This could include educational programs for expectant mothers, streamlined workflows for routine visits, and strong coordination with obstetrics and gynecology services.

3. **Financial Analysis**: Delve deeper into the increasing base costs to identify specific drivers. This could involve a review of supply costs, personnel expenses, or other operational factors that contribute to financial trends. Identify areas where efficiencies can be realized or where investments in technology could yield long-term savings.

4. **Payer Management**: With Medicare being the dominant payer, ensure that the billing processes are efficient and that potential reimbursement issues are proactively addressed. For other significant payers like Medicaid and Humana, look into contracting and negotiation practices to ensure favorable terms for the facility.

5. **COVID-19 Immunity Monitoring**: Continue tracking the COVID immunity rates among outpatients. This information could guide policies regarding telehealth services, in-person visitation protocols, and resource allocation for potential future surges in COVID-19 cases.

