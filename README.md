 # HR Analytics
> Human Resources Analytics - [A Microsoft Excel and SQL project](https://github.com/huldanalyst/Data-Analytics-Projects?tab=readme-ov-file#microsoft-excel-and-mysql-projects)

## Project Scope
In every organization, there is a need for resource management. HR Analytics provides insights into the management of the one of the invaluable kinds of resources- Human. This project provides valuable insights into the management of employees within an organization, analysing trends and behavioural patterns in respect with managerial services of the company. This project analyses employees’ performance and satisfaction index, turnover and recruitment of employees, as well as employees’ engagement in human resource training.


## Analysis
Data analysis was conducted in **MySQL**, with views created for the various segments of the project using the queries below:


#### Performance and Satisfaction
```
CREATE VIEW `average los per year (table)` AS 
SELECT YEAR(StartDate) year, ROUND(SUM(length_of_service)/COUNT(EmpID)) 'average length of service'
FROM employee_data
GROUP BY year
ORDER BY year;

CREATE VIEW `employee count by performance score` AS
SELECT `Performance Score`, COUNT(*) employee_count
FROM employee_data
WHERE ExitDate IS NULL
GROUP BY `Performance Score`
ORDER BY employee_count DESC;

CREATE VIEW `staff count below average current rating` AS
SELECT COUNT(CONCAT_WS(' ', FirstName, LastName)) old_staff_count, 
	(SELECT COUNT(CONCAT_WS(' ', FirstName, LastName)) FROM employee_data
	WHERE `Current Employee Rating` < (
			SELECT ROUND(AVG(`Current Employee Rating`))
			FROM employee_data)
		AND StartDate LIKE '2023%') new_staff_count
FROM employee_data
WHERE `Current Employee Rating` < (
        SELECT ROUND(AVG(`Current Employee Rating`))
        FROM employee_data)
	AND StartDate NOT LIKE '2023%';
    
CREATE VIEW `average satisfaction score by department` AS
SELECT YEAR(`Survey Date`) year, DepartmentType, ROUND(AVG(`Satisfaction Score`),2) 'average satisfaction score'
FROM employee_data 
JOIN employee_engagement_survey_data 
ON EmpID = `Employee ID`
WHERE ExitDate IS NULL
GROUP BY year, DepartmentType
ORDER BY year;
```
    
### Workforce Administration (Turnover & Recruitment)
```
CREATE VIEW `acceptance/recruitment rate per state` AS
WITH offered_count AS (
	SELECT Country, Status, COUNT(*) new_staff_count
	FROM recruitment_data
	WHERE Status = 'Offered'
	GROUP BY Country, Status
),
state_applicant_count AS (
	SELECT Country, COUNT(*) applicant_count
    FROM recruitment_data
    GROUP BY Country
)
SELECT Country, SUM(new_staff_count)/applicant_count*100 acceptance_rate
FROM offered_count
JOIN state_applicant_count
USING (Country)
GROUP BY Country
ORDER BY acceptance_rate DESC;

CREATE VIEW `employee count by termination type` AS
SELECT TerminationType, COUNT(*) employee_count
FROM employee_data
GROUP BY TerminationType;                                  #UNK (Unknown); They have not exited the company are not included in my analysis.

WITH employee_count_per_year AS (
		SELECT YEAR(StartDate) year, MONTH(StartDate) month_number, MONTHNAME(StartDate) month, COUNT(*) employee_count
		FROM employee_data 
		GROUP BY year, month_number, month
		),
	min_max AS (
		SELECT year, MIN(month_number) earliest_month, MAX(month_number) latest_month 
        FROM employee_count_per_year 
        GROUP BY year),
	exit_count_per_year AS (
		SELECT YEAR(ExitDate) year, COUNT(*) exit_count
		FROM employee_data
		WHERE ExitDate IS NOT NULL
		GROUP BY year
    )
SELECT year, earliest_month, latest_month, ROUND(SUM(exit_count)/(SUM(employee_count)/2),2) 'turnover rate'
FROM employee_count_per_year
JOIN min_max
USING (year)
JOIN exit_count_per_year
USING (year)
GROUP BY year, earliest_month, latest_month
ORDER BY year;

CREATE VIEW `yearly dismissal rate` AS
WITH employee_count_per_year AS (
		SELECT YEAR(StartDate) year, MIN(StartDate) year_beginning, MAX(StartDate) year_end, COUNT(*) employee_count
		FROM employee_data 
		GROUP BY year
		),
	involuntary_count_per_year AS (
		SELECT YEAR(StartDate) year, COUNT(TerminationType) involuntary_count
        FROM employee_data 
        WHERE TerminationType LIKE'In%'
        GROUP BY year
    )
SELECT year, involuntary_count /(SUM(employee_count)/2)*100 'dismissal rate'
FROM employee_count_per_year
JOIN involuntary_count_per_year
USING (year)
GROUP BY year
ORDER BY year;
```

### Human Resource Training (and Development)
```
CREATE VIEW `incomplete vs complete_assessed` AS
WITH training_duration_count AS (
	SELECT DISTINCT `Training Outcome`, COUNT(*) employee_count
	FROM training_and_development_data
	GROUP BY `Training Outcome`
	)
SELECT employee_count incomplete_count, (SELECT SUM(employee_count) FROM training_duration_count WHERE `Training Outcome` NOT LIKE 'In%') 'completed_assessed_count'
FROM training_duration_count
WHERE `Training Outcome` LIKE 'In%';
```

## Power Query 
To enable slicer connectivity and interactivity in Excel, a full outer join of relevant tables was performed within Excel's Power Query. This was necessary because MySQL does not natively support full outer joins, and the views created had dissimilar structures that made a UNION approach unfeasible.
