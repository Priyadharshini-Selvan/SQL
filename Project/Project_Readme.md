
# Club Member Information  
## An SQL Data Cleaning Project  

This project focuses on cleaning and restructuring the `club_member_info` dataset. The process involves multiple steps to ensure the dataset is accurate, consistent, and ready for analysis.


## Objectives  

- **Check for duplicate entries and remove them.**  
- **Remove extra spaces and/or other invalid characters.**  
- **Separate or combine values as needed.**  
- **Ensure that certain values (e.g., age, dates) are within specific ranges.**  
- **Check for outliers.**  
- **Correct incorrect spelling or inputted data.**  


## Additional Steps  

- **Add new and relevant rows or columns to the dataset.**  
- **Check for null or empty values.**  


## Initial Examination  

Before cleaning, let’s take a look at the first few rows of the data to understand its structure and identify potential issues:  

```sql
SELECT 
	*
FROM club_member_info
LIMIT 10;
```

### Results:  

| member_id | full_name              | age | maritial_status | email                     | phone        | full_address                        | job_title                         | membership_date |
|-----------|------------------------|-----|-----------------|---------------------------|--------------|-------------------------------------|-----------------------------------|-----------------|
| 1         | addie lush             | 40  | married         | alush0@shutterfly.com     | 254-389-8708 | 3226 Eastlawn Pass, Temple, Texas   | Assistant Professor               | 2013-07-31      |
| 2         | ROCK CRADICK           | 46  | married         | rcradick1@newsvine.com    | 910-566-2007 | 4 Harbort Avenue, Fayetteville, NC  | Programmer III                    | 2018-05-27      |
| 3         | ???Sydel Sharvell      | 46  | divorced        | ssharvell2@amazon.co.jp   | 702-187-8715 | 4 School Place, Las Vegas, Nevada   | Budget/Accounting Analyst I       | 2017-10-06      |
| 4         | Constantin de la cruz  | 35  |                 | co3@bloglines.com         | 402-688-7162 | 6 Monument Crossing, Omaha, NE      | Desktop Support Technician        | 2015-10-20      |
| 5         | Gaylor Redhole         | 38  | married         | gredhole4@japanpost.jp    | 917-394-6001 | 88 Cherokee Pass, NYC, NY           | Legal Assistant                   | 2019-05-29      |
| 6         | Wanda del mar          | 44  | single          | wkunzel5@slideshare.net   | 937-467-6942 | 10864 Buhler Plaza, Hamilton, Ohio  | Human Resources Assistant IV      | 2015-03-24      |
| 7         | Jo-ann Kenealy         | 41  | married         | jkenealy6@bloomberg.com   | 513-726-9885 | 733 Hagan Parkway, Cincinnati, OH   | Accountant IV                     | 2013-04-17      |
| 8         | Joete Cudiff           | 51  | separated       | jcudiff7@ycombinator.com  | 616-617-0965 | 975 Dwight Plaza, Grand Rapids, MI  | Research Nurse                    | 2014-11-16      |
| 9         | mendie alexandrescu    | 46  | single          | malexandrescu8@state.gov  | 504-918-4753 | 34 Delladonna Terrace, New Orleans  | Systems Administrator III         | 1921-03-12      |
| 10        | fey kloss              | 52  | married         | fkloss9@godaddy.com       | 808-177-0318 | 8976 Jackson Park, Honolulu, Hawaii | Chemical Engineer                 | 2014-11-05      |

---

## Cleaning Steps  

### 1. **Create a Temporary Table**  

To avoid altering the original data, create a temporary table for data cleaning:  

```sql
CREATE TABLE cleaned_club_member_info AS (
	SELECT
		member_id,
```

### 2. **Clean Full Names**  

- Remove extra spaces and special characters.  
- Convert names to lowercase.  

```sql
LOWER(TRIM(REGEXP_REPLACE(SUBSTRING_INDEX(full_name, ' ', 1), '[^a-zA-Z0-9]', ''))) AS first_name,
```

- Handle last names with multiple words (e.g., "de palma" or "de la cruz").  

```sql
CASE
    WHEN LENGTH(TRIM(REPLACE(full_name, ' ', ''))) - LENGTH(REPLACE(TRIM(full_name), ' ', '')) = 2 THEN CONCAT(SUBSTRING_INDEX(SUBSTRING_INDEX(full_name, ' ', -2), ' ', 1), ' ', SUBSTRING_INDEX(full_name, ' ', -1))
    WHEN LENGTH(TRIM(REPLACE(full_name, ' ', ''))) - LENGTH(REPLACE(TRIM(full_name), ' ', '')) = 3 THEN CONCAT(SUBSTRING_INDEX(SUBSTRING_INDEX(full_name, ' ', -3), ' ', 1), ' ', SUBSTRING_INDEX(SUBSTRING_INDEX(full_name, ' ', -2), ' ', 1), ' ', SUBSTRING_INDEX(full_name, ' ', -1))
    ELSE SUBSTRING_INDEX(full_name, ' ', -1)
END AS last_name,
```

### 3. **Clean Age Column**  

- Remove incorrect values (e.g., 3-digit ages).  
- Replace empty values with `NULL`.  

```sql
CASE
    WHEN age = '' THEN NULL
    WHEN LENGTH(age) = 3 THEN CAST(LEFT(age, 2) AS UNSIGNED)
    ELSE age
END AS age,
```

The `marital_status` column is cleaned by trimming any leading or trailing whitespace. 
If the column is empty after trimming, it will be set to `NULL`.

```sql
CASE
    WHEN TRIM(marital_status) = '' THEN NULL
    ELSE TRIM(marital_status)
END AS marital_status,
```

The `email` column is cleaned by trimming any leading or trailing whitespace. 
Since email addresses are case-insensitive, the email is converted to lowercase.

```sql
LOWER(TRIM(email)) AS member_email,
```


The `phone` column is cleaned by trimming any leading or trailing whitespace. 
If the phone number is empty or incomplete (less than 12 characters), it will be set to `NULL`.

```sql
CASE
    WHEN TRIM(phone) = '' THEN NULL
    WHEN LENGTH(TRIM(phone)) < 12 THEN NULL
    ELSE TRIM(phone)
END AS phone,
```


The `full_address` column is split into three separate fields: street address, city, and state. 
The data is converted to lowercase, and any whitespace is trimmed.

```sql
LOWER(TRIM(SUBSTRING_INDEX(full_address, ',', 1))) AS street_address,
LOWER(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(full_address, ',', 2), ',', -1))) AS city,
LOWER(TRIM(SUBSTRING_INDEX(full_address, ',', -1))) AS state,
```


The `job_title` column is cleaned by trimming any leading or trailing whitespace. 
The job titles containing Roman numerals (I, II, III, IV) are converted to corresponding level numbers and the descriptor (e.g., "Level 1"). If the job title is empty, it will be set to `NULL`.

```sql
CASE
    WHEN TRIM(LOWER(job_title)) = '' THEN NULL
    ELSE 
        CASE
            WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'i' THEN REPLACE(LOWER(job_title), ' i', ', level 1')
            WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'ii' THEN REPLACE(LOWER(job_title), ' ii', ', level 2')
            WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'iii' THEN REPLACE(LOWER(job_title), ' iii', ', level 3')
            WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'iv' THEN REPLACE(LOWER(job_title), ' iv', ', level 4')
            ELSE LOWER(TRIM(job_title))
        END 
END AS occupation,
```

The `membership_date` column contains some dates from the 1900s, which need to be updated to the 2000s. 
If the year is from the 1900s, it is changed to the corresponding year in the 2000s.

```sql
CASE 
    WHEN YEAR(STR_TO_DATE(membership_date, '%m/%d/%Y')) < 2000 
        THEN DATE_FORMAT(STR_TO_DATE(membership_date, '%m/%d/%Y'), '20%y-%m-%d')
    ELSE STR_TO_DATE(membership_date, '%m/%d/%Y')
END AS membership_date
```
Let's view the complete script.
```sql
DROP TABLE IF EXISTS cleaned_club_member_info;

CREATE TABLE cleaned_club_member_info (
    member_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    age INT,
    marital_status VARCHAR(255),
    member_email VARCHAR(255),
    phone VARCHAR(255),
    street_address VARCHAR(255),
    city VARCHAR(255),
    state VARCHAR(255),
    occupation VARCHAR(255),
    membership_date DATE
) AS
SELECT
    -- Some of the names have extra spaces and special characters. Trim excess whitespace, remove special characters 
    -- and convert to lowercase.
    LOWER(TRIM(REGEXP_REPLACE(SUBSTRING_INDEX(full_name, ' ', 1), '[^a-zA-Z0-9]', ''))) AS first_name,

    -- Some last names have multiple words ('de palma' or 'de la cruz'). Convert the string to an array to calculate its length and use a 
    -- case statement to find entries with those particular types of surnames.
   
    CASE
        WHEN LENGTH(TRIM(REPLACE(full_name, ' ', ''))) - LENGTH(REPLACE(TRIM(full_name), ' ', '')) = 2 THEN CONCAT(SUBSTRING_INDEX(SUBSTRING_INDEX(full_name, ' ', -2), ' ', 1), ' ', SUBSTRING_INDEX(full_name, ' ', -1))
        WHEN LENGTH(TRIM(REPLACE(full_name, ' ', ''))) - LENGTH(REPLACE(TRIM(full_name), ' ', '')) = 3 THEN CONCAT(SUBSTRING_INDEX(SUBSTRING_INDEX(full_name, ' ', -3), ' ', 1), ' ', SUBSTRING_INDEX(SUBSTRING_INDEX(full_name, ' ', -2), ' ', 1), ' ', SUBSTRING_INDEX(full_name, ' ', -1))
        ELSE SUBSTRING_INDEX(full_name, ' ', -1)
    END AS last_name,

    -- During data entry, some ages have an additional digit at the end. Remove the last digit when a 3-digit age value occurs.
    CASE
        -- Check if value is empty. If empty '' then change value to NULL
        WHEN age = '' THEN NULL
        -- First cast the integer to a string and test the character length.
        -- If condition is true, cast the integer to text, extract first 2 digits and cast back to numeric type.
        WHEN LENGTH(age) = 3 THEN CAST(LEFT(age, 2) AS UNSIGNED)
        ELSE age
    END AS age,

    -- Trim whitespace from marital_status column and if empty, ensure it's of null type
    CASE
        WHEN TRIM(marital_status) = '' THEN NULL
        ELSE TRIM(marital_status)
    END AS marital_status,

    -- Email addresses are necessary and this dataset contains valid email addresses. Since email addresses are case-insensitive,
    -- convert to lowercase and trim off any whitespace.
    LOWER(TRIM(email)) AS member_email,

    -- Trim whitespace from phone column and if empty or incomplete, ensure it's of null type
    CASE
        WHEN TRIM(phone) = '' THEN NULL
        WHEN LENGTH(TRIM(phone)) < 12 THEN NULL
        ELSE TRIM(phone)
    END AS phone,

    -- Members must have a full address for billing purposes. However, many members can live in the same household so address cannot be unique.
    -- Convert to lowercase, trim off any whitespace, and split the full address into individual street address, city, and state.
    LOWER(TRIM(SUBSTRING_INDEX(full_address, ',', 1))) AS street_address,
    LOWER(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(full_address, ',', 2), ',', -1))) AS city,
    LOWER(TRIM(SUBSTRING_INDEX(full_address, ',', -1))) AS state,

    -- Some job titles define a level in roman numerals (I, II, III, IV). Convert levels to numbers and add a descriptor (ex. Level 3).
    -- Trim whitespace from job title, rename to occupation and if empty convert to null type.
    CASE
        WHEN TRIM(LOWER(job_title)) = '' THEN NULL
        ELSE 
            CASE
                WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'i' THEN REPLACE(LOWER(job_title), ' i', ', level 1')
                WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'ii' THEN REPLACE(LOWER(job_title), ' ii', ', level 2')
                WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'iii' THEN REPLACE(LOWER(job_title), ' iii', ', level 3')
                WHEN LOWER(SUBSTRING_INDEX(job_title, ' ', -1)) = 'iv' THEN REPLACE(LOWER(job_title), ' iv', ', level 4')
                ELSE LOWER(TRIM(job_title))
            END 
    END AS occupation,

    -- A few members show membership_date year in the 1900's. Change the year into the 2000's.
    CASE 
        WHEN YEAR(STR_TO_DATE(membership_date, '%m/%d/%Y')) < 2000 
            THEN DATE_FORMAT(STR_TO_DATE(membership_date, '%m/%d/%Y'), '20%y-%m-%d')
        ELSE STR_TO_DATE(membership_date, '%m/%d/%Y')
    END AS membership_date

FROM club_member_info;
```

-- Let's take a look at our cleaned table data.
```sql
SELECT * FROM cleaned_club_member_info
LIMIT 10;
```

-- Results: ### **Results of the Cleaned Data (First 10 rows)**

| member_id | first_name | last_name  | age | marital_status | member_email              | phone         | street_address        | city          | state         | occupation                | membership_date |
|-----------|------------|------------|-----|----------------|---------------------------|---------------|-----------------------|---------------|---------------|---------------------------|-----------------|
| 1         | addie      | lush       | 40  | married        | alush0@shutterfly.com      | 254-389-8708  | 3226 eastlawn pass    | temple        | texas         | assistant professor       | 2013-07-31      |
| 2         | rock       | cradick    | 46  | married        | rcradick1@newsvine.com     | 910-566-2007  | 4 harbort avenue      | fayetteville  | north carolina | programmer, level 3       | 2018-05-27      |
| 3         | sydel      | sharvell   | 46  | divorced       | ssharvell2@amazon.co.jp    | 702-187-8715  | 4 school place        | las vegas     | nevada        | budget/accounting analyst | 2017-10-06      |
| 4         | constantin | de la cruz | 35  | NULL           | co3@bloglines.com          | 402-688-7162  | 6 monument crossing   | omaha         | nebraska      | desktop support technician| 2015-10-20      |
| 5         | gaylor     | redhole    | 38  | married        | gredhole4@japanpost.jp     | 917-394-6001  | 88 cherokee pass      | new york city | new york      | legal assistant           | 2019-05-29      |
| 6         | wanda      | del mar    | 44  | single         | wkunzel5@slideshare.net    | 937-467-6942  | 10864 buhler plaza    | hamilton      | ohio          | human resources assistant | 2015-03-24      |
| 7         | joann      | kenealy    | 41  | married        | jkenealy6@bloomberg.com    | 513-726-9885  | 733 hagan parkway     | cincinnati    | ohio          | accountant, level 4       | 2013-04-17      |
| 8         | joete      | cudiff     | 51  | separated      | jcudiff7@ycombinator.com   | 616-617-0965  | 975 dwight plaza      | grand rapids  | michigan      | research nurse            | 2014-11-16      |
| 9         | mendie     | alexandrescu| 46  | single         | malexandrescu8@state.gov  | 504-918-4753  | 34 delladonna terrace | new orleans   | louisiana     | systems administrator, level 3 | 2021-03-12  |
| 10        | fey        | kloss      | 52  | married        | fkloss9@godaddy.com        | 808-177-0318  | 8976 jackson park    | honolulu      | hawaii        | chemical engineer         | 2014-11-05      |

---



-- Now that the data is cleaned, let's look for any duplicate entries. What is the record count?
```sql
SELECT 
    COUNT(*) AS record_count 
FROM cleaned_club_member_info;
```

-- Results:

-- record_count
-- 2007

-- All members must have a unique email address to join. Let's try to find duplicate entries.
```sql
SELECT 
    member_email,
    COUNT(member_email)
FROM 
    cleaned_club_member_info
GROUP BY 
    member_email
HAVING 
    COUNT(member_email) > 1;
```

-- Results: ### **Duplicate Email Count**

| member_email                          | count |
|---------------------------------------|-------|
| hbradenri@freewebs.com                | 2     |
| omaccaughen1o@naver.com               | 2     |
| greglar4r@answers.com                 | 2     |
| ehuxterm0@marketwatch.com             | 3     |
| nfilliskirkd5@newsvine.com            | 2     |
| tdunkersley8u@dedecms.com             | 2     |
| slamble81@amazon.co.uk                | 2     |
| mmorralleemj@wordpress.com            | 2     |
| gprewettfl@mac.com                    | 2     |

---

-- Let's delete duplicate entries.
```sql
DELETE c1
FROM cleaned_club_member_info c1
JOIN cleaned_club_member_info c2 
ON c1.member_email = c2.member_email 
AND c1.member_id < c2.member_id;
```

-- What is the record count after deletion?
```sql
SELECT 
    COUNT(*) AS new_record_count 
FROM 
    cleaned_club_member_info;
```

-- Results:### **Final Record Count After Deletion**

| new_record_count |
|------------------|
| 1997             |






