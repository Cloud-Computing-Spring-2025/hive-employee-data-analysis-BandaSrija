# HadoopHiveHue
Hadoop , Hive, Hue setup pseudo distributed  environment  using docker compose

# Hive Employee Data Analysis

## Project Overview
This project focuses on analyzing employee data using Apache Hive. It involves loading employee and department datasets, performing various analytical queries, and extracting insights related to employee salaries, job roles, departments, and more.

## Implementation Approach
We used HiveQL queries to perform the following analytical tasks:
1. **Retrieve employees who joined after 2015**: Used date conversion functions to filter employees by their joining date.
2. **Find the average salary of employees in each department**: Used `AVG()` with `GROUP BY`.
3. **Identify employees working on the 'Alpha' project**: Filtered records using `WHERE` clause.
4. **Count the number of employees in each job role**: Used `COUNT()` with `GROUP BY`.
5. **Retrieve employees earning above the department average**: Used subqueries and `JOIN`.
6. **Find the department with the highest number of employees**: Used `COUNT()` with `ORDER BY` and `LIMIT`.
7. **Exclude employees with null values**: Filtered using `IS NOT NULL`.
8. **Join employees and departments tables to include department location**: Used `JOIN`.
9. **Rank employees within each department based on salary**: Used `RANK()` window function.
10. **Find the top 3 highest-paid employees in each department**: Used `DENSE_RANK()`.

## Execution Steps

#### 1. Load data into a temporary table
CREATE TABLE temp_employees (
    emp_id INT,
    name STRING,
    age INT,
    job_role STRING,
    salary DOUBLE,
    project STRING,
    join_date STRING,
    department STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;


LOAD DATA INPATH '/user/hue/employees.csv' INTO TABLE temp_employees;


#### 2. Create a partitioned table for employees
CREATE TABLE employees (
    emp_id INT,
    name STRING,
    age INT,
    job_role STRING,
    salary DOUBLE,
    project STRING,
    join_date STRING
)
PARTITIONED BY (department STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS PARQUET;


#### 3. Move data to the partitioned table
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;


INSERT INTO TABLE employees PARTITION(department)
SELECT emp_id, name, age, job_role, salary, project, join_date, department FROM temp_employees;


#### 4. Retrieve all employees who joined after 2015
SELECT * FROM employees 
WHERE year(FROM_UNIXTIME(UNIX_TIMESTAMP(join_date, 'yyyy-MM-dd'))) > 2015;


#### 5. Find the average salary of employees in each department
SELECT department, AVG(salary) AS avg_salary FROM employees GROUP BY department;


#### 6. Identify employees working on the 'Alpha' project
SELECT * FROM employees WHERE project = 'Alpha';


#### 7. Count the number of employees in each job role
SELECT job_role, COUNT(*) AS emp_count FROM employees GROUP BY job_role;


#### 8. Retrieve employees whose salary is above the average salary of their department
SELECT e1.* FROM employees e1
JOIN (SELECT department, AVG(salary) AS avg_salary FROM employees GROUP BY department) e2
ON e1.department = e2.department WHERE e1.salary > e2.avg_salary;


#### 9. Find the department with the highest number of employees
SELECT department, COUNT(*) AS emp_count FROM employees
GROUP BY department ORDER BY emp_count DESC LIMIT 1;


#### 10. Exclude employees with null values
SELECT * FROM employees WHERE emp_id IS NOT NULL AND name IS NOT NULL AND age IS NOT NULL 
AND job_role IS NOT NULL AND salary IS NOT NULL AND project IS NOT NULL AND join_date IS NOT NULL 
AND department IS NOT NULL;



#### 11. Join employees and departments tables to display employee details with department location
CREATE TABLE departments (
    dept_id INT,
    department_name STRING,
    location STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;


LOAD DATA INPATH '/user/hue/departments.csv' INTO TABLE departments;


SELECT e.*, d.location FROM employees e
JOIN departments d ON e.department = d.department_name;


#### 12. Rank employees within each department based on salary
SELECT emp_id, name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
FROM employees;


#### 13. Find the top 3 highest-paid employees in each department
```sql
SELECT * FROM (
    SELECT emp_id, name, department, salary,
           DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
    FROM employees
) ranked WHERE rank <= 3;
```

4. **Create and Load Departments Table**
    ```sql
    CREATE TABLE departments (
        dept_id INT,
        department_name STRING,
        location STRING
    )
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    STORED AS TEXTFILE;

    LOAD DATA INPATH '/user/hue/departments.csv' INTO TABLE departments;
    ```

### Running Queries
To execute any query, use:
```sql
hive -e "<SQL_QUERY>"
```
For example, to get employees who joined after 2015:
```sql
hive -e "SELECT * FROM employees WHERE year(FROM_UNIXTIME(UNIX_TIMESTAMP(join_date, 'yyyy-MM-dd'))) > 2015;"
```

## Challenges Faced
1. **Date Parsing Issue**: `TO_DATE(join_date, 'yyyy-MM-dd')` gave errors, resolved by using `FROM_UNIXTIME(UNIX_TIMESTAMP(join_date, 'yyyy-MM-dd'))`.
2. **Dynamic Partitioning**: Required enabling dynamic partition settings before inserting data.
3. **Data Format Issues**: Ensured CSV files were properly formatted and consistent before loading.

## Sample Input and Output
### Sample Input (employees.csv)
```
emp_id,name,age,job_role,salary,project,join_date,department
101,John Doe,30,Software Engineer,90000,Alpha,2016-06-15,IT
102,Jane Smith,28,Data Analyst,75000,Beta,2018-09-23,Finance
```

### Expected Output (Employees who joined after 2015)
```
emp_id | name       | age | job_role         | salary | project | join_date  | department
101    | John Doe   | 30  | Software Engineer | 90000  | Alpha   | 2016-06-15 | IT
102    | Jane Smith | 28  | Data Analyst      | 75000  | Beta    | 2018-09-23 | Finance
```


