# pgsql-http_examples
Example uses for PostgreSQL HTTP Client - https://github.com/pramsey/pgsql-http 

There is little infomation about this extension as I do not think people realise how useful it is, hopefully these examples will give other people ideas what can be done.


## Grab cvs file from a webserver

```
SELECT content FROM http_get('https://cdn.wsform.com/wp-content/uploads/2018/09/country.csv');
```
```
                      content                      
---------------------------------------------------
 Name,Code                                        +
 Afghanistan,AF                                   +
 Aland Islands,AX                                 +
 Albania,AL                                       +
 ...
 Yemen,YE                                         +
 Zambia,ZM                                        +
 Zimbabwe,ZW
(1 row)
```

Not very useful as is just a single record with a big text field.  

Using the csv_parse function from https://github.com/rin-nas/postgresql-patterns-library/blob/master/functions/csv_parse.sql lets make it more useful:

```
WITH parsed_data AS (
  SELECT csv_parse(content) AS csv_data FROM http_get('https://cdn.wsform.com/wp-content/uploads/2018/09/country.csv')
)
SELECT csv_data[1] AS country, csv_data[2] AS code FROM parsed_data;
```
```
                   country                    | code 
----------------------------------------------+------
 Afghanistan                                  | AF
 Aland Islands                                | AX
 Albania                                      | AL
 ...
 Yemen                                        | YE
 Zambia                                       | ZM
 Zimbabwe                                     | ZW
(249 rows)
```

Much more useful, as with all examples queries could include joins to other tables and views in a database.


## Using with a json rest api

Using https://covid19-api.com, lets first take a look at the fields in the data:

```
WITH restapidata AS (
   SELECT json_array_elements(content::json) AS jsondata FROM http_get('https://covid19-api.com/country/all?format=json')
)
SELECT DISTINCT json_object_keys(jsondata) AS myfields FROM restapidata ORDER BY myfields;
```
```
  myfields  
------------
 code
 confirmed
 country
 critical
 deaths
 lastChange
 lastUpdate
 latitude
 longitude
 recovered
(10 rows)
```

If you run  ```curl -X GET "https://covid19-api.com/country/all?format=json" -H  "accept: application/json"``` you can see what postgres is fetching.


Looking at some data:

```
WITH restdata AS (
   SELECT json_array_elements(content::json) AS d FROM http_get('https://covid19-api.com/country/all?format=json')
)

SELECT d->>'country' AS country, d->>'code' AS code, d->>'lastUpdate' AS last_update,
       d->>'confirmed' AS confirmed, d->>'recovered' AS recovered, d->>'deaths' AS deaths
FROM restdata ORDER BY d->>'lastUpdate' LIMIT 25;
```
```
                   country                    | code |        last_update        | confirmed | recovered | deaths 
----------------------------------------------+------+---------------------------+-----------+-----------+--------
 Jersey                                       | JE   | 2020-04-30T16:45:00+02:00 | 245       | 0         | 12
 Guernsey                                     | GG   | 2020-04-30T16:45:00+02:00 | 239       | 73        | 9
 Guam                                         | GU   | 2020-05-17T15:00:00+02:00 | 154       | 0         | 5
 U.S. Virgin Islands                          | VI   | 2020-05-17T16:00:00+02:00 | 69        | 0         | 6
 Puerto Rico                                  | PR   | 2020-05-17T17:00:00+02:00 | 2589      | 0         | 122
 Kosovo* (disputed teritory)                  |      | 2022-02-03T14:00:02+01:00 | 161484    | 40989     | 2990
 Saint Helena, Ascension and Tristan da Cunha | SH   | 2022-02-03T14:00:02+01:00 | 2         | 2         | 0
 Angola                                       | AO   | 2022-02-03T14:00:04+01:00 | 98267     | 95554     | 1895
 Andorra                                      | AD   | 2022-02-03T14:00:04+01:00 | 36315     | 33562     | 146
 Albania                                      | AL   | 2022-02-03T14:00:04+01:00 | 261270    | 240642    | 3362
 Armenia                                      | AM   | 2022-02-03T14:00:04+01:00 | 379266    | 343714    | 8065
 Aruba                                        | AW   | 2022-02-03T14:00:04+01:00 | 33146     | 32694     | 193
 Australia                                    | AU   | 2022-02-03T14:00:04+01:00 | 2644750   | 2323022   | 3987
 Austria                                      | AT   | 2022-02-03T14:00:04+01:00 | 1959017   | 1614379   | 14167
 Argentina                                    | AR   | 2022-02-03T14:00:04+01:00 | 8472848   | 7864647   | 121834
 Antigua and Barbuda                          | AG   | 2022-02-03T14:00:04+01:00 | 6732      | 6318      | 127
 Afghanistan                                  | AF   | 2022-02-03T14:00:04+01:00 | 164727    | 146768    | 7420
 Bangladesh                                   | BD   | 2022-02-03T14:00:04+01:00 | 1824180   | 1575137   | 28461
 Barbados                                     | BB   | 2022-02-03T14:00:04+01:00 | 45897     | 34901     | 282
 Belarus                                      | BY   | 2022-02-03T14:00:04+01:00 | 753495    | 742762    | 6099
 Belgium                                      | BE   | 2022-02-03T14:00:04+01:00 | 3229629   | 2095872   | 29132
 Belize                                       | BZ   | 2022-02-03T14:00:04+01:00 | 52775     | 44161     | 629
 Benin                                        | BJ   | 2022-02-03T14:00:04+01:00 | 26498     | 25506     | 163
 Bermuda                                      | BM   | 2022-02-03T14:00:04+01:00 | 10793     | 9751      | 117
 Bahrain                                      | BH   | 2022-02-03T14:00:04+01:00 | 390602    | 337175    | 1409
(25 rows)
```

Do some calculations on it:

```
WITH restdata AS (
   SELECT json_array_elements(content::json) AS d FROM http_get('https://covid19-api.com/country/all?format=json')
)

SELECT SUM(CAST(d->>'confirmed' AS integer)) AS total_confirmed, 
       SUM(CAST(d->>'recovered' AS integer)) AS total_recovered, 
       SUM(CAST(d->>'deaths' AS integer))    AS total_deaths
FROM restdata;
```
```
 total_confirmed | total_recovered | total_deaths 
-----------------+-----------------+--------------
       387378481 |       307267272 |      6336161
(1 row)
```

## Join data from different sources

We can get fancy and use postgres to join the data from the cvs and json datasets together.

Note: In country_full.csv - csv_data[2] is the code and cvs_data[6] is the continent.

```
WITH restdata AS (
   SELECT json_array_elements(content::json) AS d FROM http_get('https://covid19-api.com/country/all?format=json')
),
countrydata AS (
  SELECT csv_parse(content) AS csv_data FROM http_get('https://cdn.wsform.com/wp-content/uploads/2018/09/country_full.csv')
)
SELECT countrydata.csv_data[6] AS continent,
       COUNT(restdata.d->>'country') AS country_count,
       SUM(CAST(d->>'confirmed' AS integer)) AS total_confirmed, 
       SUM(CAST(d->>'recovered' AS integer)) AS total_recovered, 
       SUM(CAST(d->>'deaths' AS integer))    AS total_deaths
FROM restdata 
JOIN countrydata ON (restdata.d->>'code' = countrydata.csv_data[2])
GROUP BY continent
HAVING  COUNT(restdata.d->>'country') > 1
ORDER BY continent;
```
```
 continent | country_count | total_confirmed | total_recovered | total_deaths 
-----------+---------------+-----------------+-----------------+--------------
 Africa    |            60 |        11159234 |         9952687 |       240442
 Americas  |            57 |       139380190 |       100646266 |      2549995
 Asia      |            51 |       101977101 |        94226233 |      1302306
 Europe    |            51 |       131955858 |        99956943 |      2234155
 Oceania   |            29 |         2841370 |         2484645 |         6394
(5 rows)
```


## Materialized views

To speed things up with large datasets, reduce network traffic or if you have a rate limit on api calls it may be useful to make a materialized view.

```
CREATE MATERIALIZED VIEW coviddata AS
   SELECT json_array_elements(content::json) AS d FROM http_get('https://covid19-api.com/country/all?format=json');
```

Now we can query the materialized view and not connect to the rest api server or other web host.

```
SELECT d->>'country' AS country FROM coviddata ORDER BY d->>'country' LIMIT 10;
```
```
       country       
---------------------
 Afghanistan
 Albania
 Algeria
 American Samoa
 Andorra
 Angola
 Anguilla
 Antarctica
 Antigua and Barbuda
 Argentina
(10 rows)
```

Remeber to refresh the  materialized view when its data needs to be updated:
```
REFRESH MATERIALIZED VIEW coviddata;
```


## XML / RSS feeds

The xml document must be valid or xpath will error. 

```
WITH rawxml AS (
   SELECT content::xml FROM http_get('https://feeds.theguardian.com/theguardian/science/rss')
), 
unnestdata AS (  
   SELECT unnest(xpath('//item/title/text()', content)) as title,
          unnest(xpath('//item/link/text()', content)) as link
   FROM rawxml
)
SELECT substring(title::text,1,35) AS title,
       link::text AS link
FROM unnestdata LIMIT 10;
```
```
                title                |                                                                      link                                                                      
-------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------
 International Space Station will pl | https://www.theguardian.com/science/2022/feb/02/international-space-station-will-plummet-to-a-watery-grave-in-2030
 The great gaslighting: how Covid lo | https://www.theguardian.com/society/2022/feb/03/long-covid-fight-recognition-gaslighting-pandemic
 German researchers to breed pigs fo | https://www.theguardian.com/science/2022/feb/03/german-researchers-to-breed-pigs-for-human-heart-transplants
 First patients of pioneering CAR T- | https://www.theguardian.com/society/2022/feb/02/first-patients-pioneering-car-t-cell-therapy-cured-of-cancer
 Super corals: the race to save the  | https://www.theguardian.com/environment/gallery/2022/feb/03/super-corals-the-race-to-save-the-worlds-reefs-from-the-climate-crisis-in-pictures
 Antarcticaâs âDoomsday Glacierâ kee | https://www.theguardian.com/world/2022/feb/02/antarctica-doomsday-glacier-scientists-frustrated-iceberg
 Are we getting any closer to unders | https://www.theguardian.com/science/audio/2022/feb/03/are-we-getting-any-closer-to-understanding-long-covid
 Covid vaccine hesitancy could be li | https://www.theguardian.com/society/2022/feb/01/covid-vaccine-hesitancy-could-be-linked-childhood-trauma-research
 Rise in Covid cases in England as r | https://www.theguardian.com/world/2022/jan/31/spike-in-covid-cases-in-england-as-reinfections-included-for-first-time
 Joe Roganâs Covid claims: what does | https://www.theguardian.com/culture/2022/jan/31/joe-rogan-covid-claims-what-does-the-science-actually-say
(10 rows)
```

 I have tried to use xpath to extract a table from various webpages but most are not valid and found ```SELECT xml_is_well_formed(content) FROM rawxml;``` is a useful query when trying to parse xml documents.


