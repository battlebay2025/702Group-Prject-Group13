--Clean Data
--Clear Negative values
DELETE FROM covid_19_data
WHERE Confirmed < 0
or Deaths < 0 
or Recovered < 0;

--Initial harmonization of some country names through simple manual verification
UPDATE country_vaccinations
SET country = 'US'
WHERE country = 'United States';

UPDATE country_vaccinations
SET country = 'UK'
WHERE country = 'United Kingdom';

UPDATE country_vaccinations_by_manufacturer
SET location = 'US'
WHERE location = 'United States';

UPDATE country_vaccinations_by_manufacturer
SET location = 'UK'
WHERE location = 'United Kingdom';

UPDATE country_vaccinations
SET country = 'China'
WHERE country IN ('Hong Kong', 'Macau', 'Taiwan');

UPDATE country_vaccinations_by_manufacturer
SET location = 'China'
WHERE location IN ('Hong Kong', 'Macau', 'Taiwan');

UPDATE covid_19_data
SET "Country/Region" = 'China'
WHERE "Country/Region" IN ('Hong Kong', 'Macau', 'Taiwan', 'Mainland China');

UPDATE covid_19_data
SET "Province/State" = NULL
WHERE "Province/State" = 'None' ;

UPDATE covid_19_data
SET "Province/State" = NULL
WHERE "Province/State" = 'Unknown';

UPDATE covid_19_data
SET "Country/Region" = 'St. Martin'
WHERE "Country/Region" LIKE '%St. Martin%';

UPDATE covid_19_data
SET ObservationDate = REPLACE(ObservationDate, '/', '-');

UPDATE covid_19_data
SET ObservationDate = SUBSTR(ObservationDate, 7, 4) || '-' || SUBSTR(ObservationDate, 1, 2) || '-' || SUBSTR(ObservationDate, 4, 2);

--Check Problems
--Check the number of data that covid_19_data table and country_vaccination table can match
SELECT COUNT(*)
FROM covid_19_data AS TD
JOIN country_vaccinations AS CV 
ON TD."Country/Region" = CV.country 
AND TD.ObservationDate = CV.date;

SELECT COUNT(*) FROM covid_19_data;

--Identify country names that do not match in the covid_19_data table and country_vaccination table
SELECT DISTINCT T1."Country/Region"
FROM covid_19_data AS T1
LEFT JOIN country_vaccinations AS T2 ON T1."Country/Region" = T2.country
WHERE T2.country IS NULL;

--Harmonization of country names
UPDATE covid_19_data
SET "Country/Region" = 'The Bahamas'
WHERE "Country/Region" = 'Bahamas, The';

UPDATE covid_19_data
SET "Country/Region" = 'The Gambia'
WHERE "Country/Region" = 'Gambia, The';

UPDATE covid_19_data
SET "Country/Region" = 'Ivory Coast'
WHERE "Country/Region" = 'Cote d''Ivoire';

UPDATE covid_19_data
SET "Country/Region" = 'Republic of the Congo'
WHERE "Country/Region" = 'Congo (Brazzaville)' OR "Country/Region" = 'Republic of the Congo';

UPDATE covid_19_data
SET "Country/Region" = 'Democratic Republic of the Congo'
WHERE "Country/Region" = 'Congo (Kinshasa)';

UPDATE covid_19_data
SET "Country/Region" = 'Myanmar'
WHERE "Country/Region" = 'Burma';

UPDATE covid_19_data
SET "Country/Region" = 'Czechia'
WHERE "Country/Region" = 'Czech Republic';

UPDATE covid_19_data
SET "Country/Region" = 'Timor-Leste'
WHERE "Country/Region" = 'East Timor';

UPDATE covid_19_data
SET "Country/Region" = 'Ireland'
WHERE "Country/Region" = 'Republic of Ireland';

UPDATE covid_19_data
SET "Country/Region" = 'UK'
WHERE "Country/Region" = 'Channel Islands' OR "Country/Region" = 'North Ireland';

UPDATE covid_19_data
SET "Country/Region" = 'France'
WHERE "Country/Region" IN ('French Guiana', 'Guadeloupe', 'Martinique', 'Mayotte', 'Reunion', 'Saint Barthelemy');

UPDATE covid_19_data
SET "Country/Region" = 'US'
WHERE "Country/Region" = 'Puerto Rico' OR "Country/Region" = 'Guam';

DELETE FROM covid_19_data
WHERE "Country/Region" IN ('Diamond Princess', 'MS Zaandam', 'Others');

UPDATE covid_19_data
SET "Country/Region" = 'Palestine'
WHERE "Country/Region" IN ('occupied Palestinian territory', 'West Bank and Gaza');

UPDATE country_vaccinations
SET country = 'UK'
WHERE country IN ('England', 'Scotland', 'Wales');

UPDATE country_vaccinations
SET country = 'Timor-Leste'
WHERE country = 'Timor';

UPDATE country_vaccinations
SET country = 'UK'
WHERE country IN ('Anguilla', 'Bermuda', 'British Virgin Islands', 'Cayman Islands', 'Falkland Islands', 'Gibraltar', 'Guernsey', 'Isle of Man', 'Jersey', 'Montserrat', 'Saint Helena', 'Turks and Caicos Islands');

UPDATE country_vaccinations
SET country = 'France'
WHERE country IN ('French Polynesia', 'Wallis and Futuna');

UPDATE country_vaccinations
SET country = 'New Zealand'
WHERE country IN ('Cook Islands', 'Niue', 'Tokelau');

UPDATE country_vaccinations
SET country = 'Netherlands'
WHERE country IN ('Bonaire Sint Eustatius and Saba', 'Sint Maarten (Dutch part)');

UPDATE country_vaccinations
SET country = 'Republic of the Congo'
WHERE country = 'Congo';

UPDATE country_vaccinations
SET country = 'Democratic Republic of the Congo'
WHERE country = 'Democratic Republic of Congo';

UPDATE country_vaccinations
SET country = 'Cote d''Ivoire'
WHERE country = 'Cote d''Ivoire';

UPDATE country_vaccinations
SET country = 'Northern Cyprus'
WHERE country = 'Northern Cyprus';

--Clean Vaccine Name 
CREATE TABLE Vaccine_Link_Table (
    country TEXT,
    date TEXT,
    vaccine_name TEXT,
    PRIMARY KEY (country, date, vaccine_name)
);

--Use recursive CTE (Common Table Expression) to split a string containing multiple vaccine names.
WITH RECURSIVE split_vaccines(country, date, single_vaccine, remaining_string) AS (
  SELECT
    country,
    date,
    TRIM(SUBSTR(vaccines, 1, INSTR(vaccines || ',', ',') - 1)),
    SUBSTR(vaccines, INSTR(vaccines || ',', ',') + 1)
  FROM
    country_vaccinations
  UNION ALL
  SELECT
    country,
    date,
    TRIM(SUBSTR(remaining_string, 1, INSTR(remaining_string || ',', ',') - 1)),
    SUBSTR(remaining_string, INSTR(remaining_string || ',', ',') + 1)
  FROM
    split_vaccines
  WHERE
    remaining_string != ''
)
INSERT OR IGNORE INTO Vaccine_Link_Table (country, date, vaccine_name)
SELECT
    country,
    date,
    single_vaccine
FROM
    split_vaccines
WHERE
    single_vaccine != '';



--Create Dim Table
CREATE TABLE Location(
LocationID INTEGER PRIMARY KEY AUTOINCREMENT,
CountryName TEXT,
Province_State TEXT
);

CREATE TABLE "Time" (
    time_id INTEGER PRIMARY KEY,
    date TEXT UNIQUE,
    year INTEGER,
    month INTEGER,
    day INTEGER);

CREATE TABLE Vaccine(
VaccineID INTEGER PRIMARY KEY AUTOINCREMENT,
VaccineName TEXT,
vaccineType TEXT
);


--Insert location data
INSERT INTO Location (CountryName, Province_State)
SELECT
    TRIM(Country) AS CountryName,
    NULL AS Province_State
FROM
    country_vaccinations
WHERE Country IS NOT NULL AND Country != ''
UNION
SELECT
    TRIM(location) AS CountryName,
    NULL AS Province_State
FROM
    country_vaccinations_by_manufacturer
WHERE location IS NOT NULL AND location != ''
UNION
SELECT
    TRIM("Country/Region") AS CountryName,
    TRIM("Province/State") AS Province_State
FROM
    covid_19_data
WHERE "Country/Region" IS NOT NULL AND "Country/Region" != '';


--Insert time data
WITH RECURSIVE DateSeries(date) AS (
    SELECT MIN(date_val) FROM (
        SELECT MIN(date(ObservationDate)) as date_val FROM covid_19_data
        UNION ALL
        SELECT MIN(date) as date_val FROM country_vaccinations_by_manufacturer
        UNION ALL
        SELECT MIN(date) as date_val FROM country_vaccinations
    )
    UNION ALL
    SELECT date(date, '+1 day')
    FROM DateSeries
    WHERE date < (
        SELECT MAX(date_val) FROM (
            SELECT MAX(date(ObservationDate)) as date_val FROM covid_19_data
            UNION ALL
            SELECT MAX(date) as date_val FROM country_vaccinations_by_manufacturer
            UNION ALL
            SELECT MAX(date) as date_val FROM country_vaccinations
        )
    )
)
INSERT INTO "Time" (date, year, month, day)
SELECT DISTINCT
    date,
    CAST(strftime('%Y', date) AS INTEGER),
    CAST(strftime('%m', date) AS INTEGER),
    CAST(strftime('%d', date) AS INTEGER)
FROM
    DateSeries;


--Insert vaccine data
INSERT INTO Vaccine (VaccineName, vaccineType)
SELECT DISTINCT(vaccine),
CASE
        WHEN vaccine = 'Pfizer/BioNTech' THEN 'mRNA'
        WHEN vaccine = 'Moderna' THEN 'mRNA'
        WHEN vaccine = 'Oxford/AstraZeneca' THEN 'Viral Vector'
        WHEN vaccine = 'Johnson&Johnson' THEN 'Viral Vector'
        WHEN vaccine = 'Sputnik V' THEN 'Viral Vector'
        WHEN vaccine = 'Sinopharm/Beijing' THEN 'Inactivated Virus'
        ELSE 'Other'
    END AS vaccineType
FROM country_vaccinations_by_manufacturer;

--Create Bridge table which contains the daily infections and daily deathe data
--use MAX() to ensure there would be no negative values
CREATE TABLE TEMP_daily AS
SELECT
    "Country/Region",
    "Province/State",
    ObservationDate,
    MAX(0, Confirmed - LAG(Confirmed, 1, 0) OVER 
	(PARTITION BY "Country/Region", "Province/State" ORDER BY "ObservationDate")) AS Daily_Infections,
    MAX(0, Deaths - LAG(Deaths, 1, 0) OVER 
	(PARTITION BY "Country/Region", "Province/State" ORDER BY "ObservationDate")) AS Daily_Deaths
FROM
    covid_19_data
ORDER BY
    ObservationDate, "Country/Region";


	
--Create Factor Table
CREATE TABLE Fact_COVID_Metrics (
    fact_id INTEGER PRIMARY KEY,
    time_id INTEGER,
    LocationID INTEGER,
    VaccineID INTEGER, 
    Daily_Infections INTEGER, 
    Daily_Deaths INTEGER, 
    Daily_Doses_of_This_Vaccine INTEGER, 
    FOREIGN KEY (time_id) REFERENCES "Time"(time_id),
    FOREIGN KEY (LocationID) REFERENCES "Location"(LocationID),
    FOREIGN KEY (VaccineID) REFERENCES "Vaccine"(VaccineID));

--Insert data into factor table
--also use MAX() to ensure there would be no negative values in total vaccinations
INSERT INTO Fact_COVID_Metrics (
    time_id,
    LocationID,
    VaccineID,
    Daily_Infections,
    Daily_Deaths,
    Daily_Doses_of_This_Vaccine)
WITH Daily_Vaccine_Doses AS (
    SELECT
        location,
        date,
        vaccine,
       MAX(0, total_vaccinations - LAG(total_vaccinations, 1, 0) OVER (PARTITION BY location, vaccine ORDER BY date)) AS daily_doses
    FROM
        country_vaccinations_by_manufacturer)
SELECT
    DD.time_id,
    DL.LocationID,
    DV.VaccineID,
    TD.Daily_Infections,
    TD.Daily_Deaths,
    DVD.daily_doses
FROM
    Daily_Vaccine_Doses AS DVD
LEFT JOIN TEMP_daily AS TD ON DVD.location = TD."Country/Region" AND DVD.date = TD.ObservationDate
LEFT JOIN "Time" AS DD ON DVD.date = DD."Date"
LEFT JOIN "Location" AS DL ON DVD.location = DL.CountryName
LEFT JOIN "Vaccine" AS DV ON DVD.vaccine = DV.VaccineName
WHERE
    DL.LocationID IS NOT NULL AND DV.VaccineID IS NOT NULL;


--Clean the null in factor table
UPDATE Fact_COVID_Metrics
SET Daily_Infections = 0
WHERE Daily_Infections IS NULL;

UPDATE Fact_COVID_Metrics
SET Daily_Deaths = 0
WHERE Daily_Deaths IS NULL;

-- Research Questions
--Q1
-- vaccine technology deployment and changes in infection rates in the US during 2021.
SELECT T.month, V.vaccineType, F.Daily_Doses_of_This_Vaccine AS TotalDosesAdministered, AVG(F.Daily_Infections) AS AverageDailyInfections
FROM Fact_COVID_Metrics AS F
JOIN "Location" AS L ON F.LocationID = L.LocationID
JOIN "Time" AS T ON F.time_id = T.time_id
JOIN "Vaccine" AS V ON F.VaccineID = V.VaccineID
WHERE L.CountryName = 'US' 
AND T.year = 2021
AND T.month <= 5
GROUP BY T.month,V.vaccineType
ORDER BY V.vaccineType, T.month;
	
	
--Q2
-- This query prepares a dataset to analyze the correlation between vaccine diversity
SELECT L.CountryName, COUNT(DISTINCT F.VaccineID) AS NumberOfVaccineTypes, SUM(F.Daily_Deaths) AS TotalDeaths
FROM Fact_COVID_Metrics AS F
JOIN Location L 
ON F.LocationID = L.LocationID
JOIN "Time" T 
ON F.time_id = T.time_id
WHERE T.year = 2021
AND F.Daily_Deaths IS NOT NULL
GROUP BY L.CountryName
HAVING SUM(F.Daily_Deaths) > 0;
	
--Q3
-- This query extact the daily infections
-- for each Australian state in 2020 to analyze the pandemic's trajectory.
WITH Daily_Metrics_AU AS (
    -- First, calculate the daily new infections and deaths from the cumulative data
    SELECT
        "ObservationDate" AS date,
        "Province/State" AS state,
        Daily_Infections
    FROM
        TEMP_daily
    WHERE
        "Country/Region" = 'Australia')
SELECT
    date,
    state,
    Daily_Infections
FROM
    Daily_Metrics_AU
WHERE
    STRFTIME('%Y', date) = '2020';
