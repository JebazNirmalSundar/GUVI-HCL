# Hospital Analytics Dashboard
Creating PowerBI Dashboard For Hospital Analytics

## 📊 Project Overview
Using Excel for store and import the file to MYSQL for working on the dataset to 
changes the null values, modify the datas and more.
This Power BI dashboard was created as part of the HCL Round 2 interview process.
It provides interactive insights and business-level analysis.

## 🛠 Tools Used
- MySql
- Power BI
- Excel / CSV
- DAX

# SQL Query
-- 1. create the database
create database hospital_db;
use hospital_db;

-- 2. create patients table
create table patients (
    patient_id int primary key,
    age int,
    gender varchar(10),
    insurance_type varchar(20)
);

-- 3. create admissions table
create table admissions (
    admission_id int primary key,
    patient_id int,
    hospital_branch varchar(50),
    department varchar(50),
    admission_date date,
    discharge_date date,
    case_type varchar(20),
    outcome varchar(20),
    icu_required varchar(10), 
    readmission_flag varchar(10),
    foreign key (patient_id) references patients(patient_id)
);

-- 4. create doctors table
create table doctors (
    doctor_id varchar(10) primary key,
    department varchar(50),
    hospital_branch varchar(50),
    hours_available int,
    hours_booked int
);

-- 5. create beds table
create table beds (
    bed_id varchar(10),
    department varchar(50),
    hospital_branch varchar(50),
    date date,
    is_occupied varchar(10)
);

-- 6. create billing table
create table billing (
    admission_id int primary key,
    total_cost decimal(10, 2),
    billing_date date,
    foreign key (admission_id) references admissions(admission_id)
);

-- data verification
Select * from patients limit 5;
Select count(*) from patients;

Select * from admissions limit 5;
Select count(*) from admissions;

Select * from beds limit 5;
Select count(*) from beds;

Select * from doctors limit 5;
Select count(*) from doctors;

Select * from billing limit 5;
Select count(*) from billing;

-- checking if any important admission records have missing dates
select count(*) from admissions 
where admission_date is null or discharge_date is null;
-- checking if any patient_id appears twice 
select patient_id, count(*) 
from patients 
group by patient_id 
having count(*) > 1;
-- check date logic
Select * from admissions where discharge_date < admission_date;

-- Core Metrics (KPIs)
-- 1. Average length of stay (ALOS)
select department,avg(datediff(discharge_date, admission_date)) as ALOS_days
from admissions
group by department;
-- 2. Bed occupancy rate
select hospital_branch,(count(case when is_occupied = 'true' then 1 end) * 100.0 / count(*)) as occupancy_rate
from beds
group by hospital_branch;
-- 3. Patient admission & discharge counts
select department,
    count(admission_id) as total_admissions,
    count(discharge_date) as total_discharges
from admissions
group by department;
-- 4. Readmission rate (30-day)
select count(distinct t2.patient_id) * 100.0 / count(distinct t1.patient_id) as calculated_30_day_rate
from admissions t1
left join admissions t2 on t1.patient_id = t2.patient_id 
    and t2.admission_date > t1.discharge_date 
    and t2.admission_date <= date_add(t1.discharge_date, interval 30 day);
-- 5. Procedure volume
select department,count(admission_id) as procedure_count
from admissions group by department
order by procedure_count desc;
-- 6. Emergency vs. scheduled cases 
select department, case_type,
    count(*) as case_count
from admissions
group by department, case_type;
-- 7. Doctor utilization (% time booked)
SELECT 
    department,
    AVG(hours_booked * 100.0 / hours_available) AS avg_utilization
FROM
    doctors
GROUP BY department;
-- 8. cost per patient & billing breakdown 
select a.department,
    sum(b.total_cost) as total_revenue,
    avg(b.total_cost) as avg_cost_per_patient
from admissions a
join billing b on a.admission_id = b.admission_id
group by a.department;
-- 9. patient outcome classification
select department,outcome,
    count(*) as outcome_count
from admissions
group by department, outcome;


create view master_dashboard_view as
select 
    -- 1. identification
    a.admission_id,
    a.hospital_branch,
    a.department,
    
    -- 2. patient demographics
    p.patient_id,
    p.age,
    p.gender,
    p.insurance_type,
    
    -- 3. clinical & outcome data
    a.case_type,
    a.outcome,
    a.admission_date,
    a.discharge_date,
    datediff(a.discharge_date, a.admission_date) as length_of_stay,
    a.icu_required,
    a.readmission_flag,
    
    -- 4. financial data
    b.total_cost,
    b.billing_date
    
from admissions a
join patients p on a.patient_id = p.patient_id
left join billing b on a.admission_id = b.admission_id;

select * from master_dashboard_view;

-- 1. Trend analysis (daily, weekly, monthly)
select 
    date_format(admission_date, '%y-%m') as month,
    sum(total_cost) as total_revenue
from master_dashboard_view
group by month
order by month asc;
-- 2. Cross-department and branch comparison
select hospital_branch,department, 
    count(admission_id) as patient_count,
    avg(total_cost) as avg_billing
from master_dashboard_view
group by hospital_branch, department;
-- 3. Bottleneck identification (icu and readmission)
select department,
    sum(case when icu_required = 'true' then 1 else 0 end) as icu_total,
    sum(case when readmission_flag = 'true' then 1 else 0 end) as readmissions
from master_dashboard_view
group by department;
-- 4. Drill-down by age and insurance type
select insurance_type,
    case 
        when age < 18 then 'child'
        when age between 18 and 60 then 'adult'
        else 'senior'
    end as age_group,
    count(*) as patient_total
from master_dashboard_view
group by insurance_type, age_group;
-- 5. Staffing optimization (peak days)
select 
    date_format(admission_date, '%y-%m') as month,
    count(admission_id) as total_admissions
from master_dashboard_view
group by month;
-- 6. Revenue by insurance type (drill-down)
select 
    insurance_type,
    sum(total_cost) as total_revenue,
    count(admission_id) as patient_count
from master_dashboard_view
group by insurance_type;
-- 7. Average billing per patient type
select case_type,
    round(avg(total_cost), 2) as avg_cost
from master_dashboard_view
group by case_type;
-- 8. Revenue by department (comparison)
select 
    department,
    sum(total_cost) as revenue,
    round(avg(total_cost), 2) as avg_bill_amount
from master_dashboard_view
group by department
order by revenue desc;
-- 9. Mortality and outcome analysis
select department,outcome,
    count(admission_id) as total_cases
from admissions
where outcome in ('deceased', 'transferred')
group by department, outcome;

## 📈 Key Insights
- Overall Sales & Profit
- Region-wise analysis
- Product performance
- Trend analysis using slicers

## 📂 Files
- Master data.xlsc
- Mysql
- Power BI (.pbix)
- Dataset
- Dashboard screenshots![HOSPITAL ANALYTICS DASHBOARD](https://github.com/user-attachments/assets/e15212fc-e8f3-4b4e-9f34-7d5268008ec7)


## 👤 Created By
Jebaz Nirmal Sundar
