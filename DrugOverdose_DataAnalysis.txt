
-- Note: To check the final table, please run the last query of this file.


--<< start >>--
--|------------------------------------------------
--| Query To Create a virtual table named my_ans
--|------------------------------------------------
Create view my_ans
As
With data as(
-- Get patients died during medication using 'patient','medications' tables.
select  distinct patient
        ,birthdate
        ,deathdate
        ,min(date(start)) over(partition by patient) as start_dt
        ,max(date(stop)) over(partition by patient) as stop_dt
        ,case when deathdate = max(date(stop)) over(partition by patient) then 1 
                  else 0 
         end as DEATH_AT_VISIT_IND
from medications
inner join patients
        on patients.id = medications.patient
order by patient),
 
data1 as(
-- Calculate the total count of current medications against each patient
select patient
        ,count(description) as COUNT_CURRENT_MEDS
from encounters
group by patient),
 
data2 as(
-- To show if patient is having opioid medicines:'Hydromorphone 325 MG', 'Fentanyl 100 MCG', 'Oxycodone-acetaminophen 100ML'
select distinct patient
        ,case when description in ('Hydromorphone 325 MG', 'Fentanyl 100 MCG', 'Oxycodone-acetaminophen 100ML')
                  then 1
                  else 0
        end as CURRENT_OPIOID_IND
from medications
order by patient),
 
data3 as(
-- To show the first readmission date means 2nd admission date of patients.
Select distinct id
        ,patient
        ,description
        ,start as FIRST_READMISSION_DATE 
from (select id
                ,start
                ,stop
                ,patient
                ,description
                ,row_number() over(partition by patient order by start asc) as row_number
                 from encounters
                 where description like '%admi%')encounters 
where row_number = 2 order by patient),
 
data4 as(
-- To show the patient encounter's 1st date
select distinct patient
        ,start as HOSPITAL_ENCOUNTER_DATE
from (select id
          ,start
          ,stop
          ,patient
          ,description
          ,row_number() over(partition by patient order by start asc) as row_number
          from encounters)encounters
where row_number = 1
order by patient),


data5 as(
-- To show 1, if the visit resulted in a subsequent drug overdose readmission within 90 days, 0 if not.
Select distinct patient
        ,start
        ,stop
        ,description
        ,case when diffdays < 90 then 1
        else 0
        end as READMISSION_90_DAY_IND
from (Select start
          ,stop
          ,patient
          ,description
          ,Extract(day from start::timestamp - previous_stop_date::timestamp) as diffdays
         from (select start
                   ,stop
                   ,patient
                   ,description
                   ,LAG(stop,1,Null) over (partition by patient order by start, stop) as previous_stop_date
                   from encounters
                   where description like '%admi%')Lagselect 
          where previous_stop_date is not null)Lagselect),


data6 as(
-- To show 1, if the visit resulted in a subsequent drug overdose readmission within 30 days, 0 if not overdose encounter, 0 if not
Select distinct patient
        ,start
        ,stop
        ,description
        ,case when diffdays < 30 then 1
        else 0
        end as READMISSION_30_DAY_IND
from(Select distinct patient
         ,start
         ,stop
         ,description
         ,Extract(day from start::timestamp - previous_stop_date::timestamp) as diffdays
        from (select distinct patient
                  ,start
                  ,stop
                  ,description
                  ,LAG(stop,1,Null) over(partition by patient order by start, stop) as previous_stop_date
                  from encounters
                  where description like '%admi%'order by patient) Lagselect
         where previous_stop_date is not null)Lagselect)
--|-----------------------------------------------------------------------------------------------------------------------
--| This query combines all of the above queries using join and refelects only necessary columns asked in the assignment.
--|-----------------------------------------------------------------------------------------------------------------------
select distinct data2.patient as PATIENT_ID
        ,data3.id as ENCOUNTER_ID
         ,data4.HOSPITAL_ENCOUNTER_DATE
         ,date_part('year',Age(data.start_dt,data.birthdate)) as AGE_At_VISIT
         ,data.DEATH_AT_VISIT_IND
         ,data1.COUNT_CURRENT_MEDS
         ,data2.CURRENT_OPIOID_IND
          ,data5.READMISSION_90_DAY_IND
        ,data6.READMISSION_30_DAY_IND
         ,data3.FIRST_READMISSION_DATE
from data6
join data5
        on data5.patient = data6.patient
join data4
         on data4.patient = data5.patient
join data3
        on data3.patient = data4.patient
join data2
        on data2.patient = data3.patient
join data1
        on data1.patient = data2.patient
join data
        on data.patient = data1.patient


--<< end >>--
--|------------------------------------------------
--| Final table (Query to get the virtual table)
--|------------------------------------------------
select distinct * from my_ans
where HOSPITAL_ENCOUNTER_DATE > '1999-07-15'
        and Age_At_visit between 18 and 35
order by patient_id

--=================================================
