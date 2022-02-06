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



## Working with CouchDB 

This scales pretty well for example the couchdb database being used as an example is a bit less than 1Gb and contains over 250,000 documents.

Need to set some options or it will timeout plus couchdb wants a user and password, these get reset if you reconnect to postgres.

```
SELECT * FROM http_set_curlopt('CURLOPT_USERPWD', 'admin:passwd');
SELECT * FROM http_set_curlopt('CURLOPT_TIMEOUT', '3600');
SELECT * FROM http_list_curlopt();
```
```
     curlopt     |     value      
-----------------+----------------
 CURLOPT_TIMEOUT | 3600
 CURLOPT_USERPWD | admin:password
(2 rows)
```


Lets grab all the couchdb docs and put them into a materialized view with a copy of the document id field "key" and the whole document in jsonb field called "doc".

```
CREATE MATERIALIZED VIEW couchdata AS
 
   SELECT * FROM json_to_recordset(
     (
        SELECT (content::json->>'rows')::json 
        FROM http_get('http://192.168.3.25:5984/articles/_all_docs?include_docs=true')
      )
 
   ) AS couchdata (key text, doc jsonb);
```

Timed output from psql so you can see its pretty fast - not sure how any script to pull the data and insert it could be any faster.

```
=# CREATE MATERIALIZED VIEW couchdata AS
-#  
-#    SELECT * FROM json_to_recordset(
(#      (
(#         SELECT (content::json->>'rows')::json 
(#         FROM http_get('http://192.168.3.25:5984/articles/_all_docs?include_docs=true')
(#       )
(#  
(#    ) AS couchdata (key text, doc jsonb);
SELECT 295276
Time: 87150.647 ms (01:27.151)
```

```
SELECT count(*) FROM couchdata;
 count  
--------
 295276
(1 row)
```

Lets see what fields there are:

```SELECT DISTINCT jsonb_object_keys(doc) AS myfields FROM couchdata ORDER BY myfields;```
```
  myfields   
-------------
 _id
 _rev
 description
 feedName
 icon
 indexes
 language
 link
 pubDate
 pubDateTS
 read
 starred
 tags
 title
 views
(15 rows)
```

And take a look at some doc titles:

```SELECT doc->>'title' as title FROM couchdata LIMIT 5;```
```
                                                     title                                                      
-------------------------------------------------------------------------------------------------------------------
 Revolutionary lung cancer drug made available on NHS in England
 Microsoft stands on shore as tablet-laden boat sails away
 Japan Eyes World's Fastest-Known Supercomputer, To Spend Over $150M On It
 In a Throwback To the '90s, NTFS Bug Lets Anyone Hang Or Crash Windows 7, 8.1
 GNOME's New Human Interface Guidelines Now Official
(5 rows)
```

And again whenever you need an updated version of the data you can just refresh the view:

```REFRESH MATERIALIZED VIEW couchdata;```

If you need a near realtime version of your couch data in postgres take a look at: https://github.com/sysadminmike/couch-to-postgres
 


## Manipulating incomming data 

Using couchdb as an example to add/modify data for example extract the title from the doc json or make it easier to search.

```
CREATE MATERIALIZED VIEW couchsearch AS   
  WITH cj AS (
   SELECT * FROM json_to_recordset(
     (
        SELECT (content::json->>'rows')::json 
        FROM http_get('http://192.168.3.25:5984/articles/_all_docs?include_docs=true')
      )) AS couchdata (key text, doc jsonb)
   )
   SELECT key, doc->>'title'::text AS title, 
         setweight(to_tsvector('pg_catalog.english', coalesce(doc->>'title','')), 'A') || 
         setweight(to_tsvector('pg_catalog.english', coalesce(doc->>'description','')), 'D') AS tsv,
         doc
   FROM cj;
```

The above took ```Time: 167910.774 ms (02:47.911)``` to run which isnt bad to grab nearly 300k documents and create a ts vector for them, there were a few warnings from to_tsvector:
```
NOTICE:  word is too long to be indexed
DETAIL:  Words longer than 2047 characters are ignored.
```

Searching takes under a second on a two word query:
```
SELECT title,
       ts_rank_cd(tsv, to_tsquery('FreeBSD & Postgres')) AS rank
FROM couchsearch 
WHERE tsv @@ to_tsquery('FreeBSD & Postgres')
ORDER BY rank DESC LIMIT 10;
```
```
                                  title                                   |     rank     
--------------------------------------------------------------------------+--------------
 Postgres in FreeBSD Jails                                                |          0.5
 This week's Postgres news                                                |   0.06458476
 A library for parsing and deparsing Postgres queries                     |  0.039215688
 psql tips                                                                |   0.01969697
 How 'RETURNING' yielded a 9x performance improvement                     |  0.018382354
 In Other BSDs for 2019/08/31                                             |  0.013636364
 Working with row level security in Postgres                              |  0.011190476
 Making Mystery-Solving Easier with `auto_explain`                        | 0.0051190476
 PostgreSQL Weekly News - January  2, 2022                                |        0.004
 SysAdmin1138: Sysadmins and risk-management                              | 0.0037037039
(10 rows)

Time: 912.203 ms
```


## POST-ing

All previous examples were just using GET.
Using couchdb json query syntax: https://docs.couchdb.org/en/3.2.0/api/database/find.html we need to POST rather than GET.
Limit of 10 is being added to request to couchdb so we can reduce the number of records before they hit postgres.

```
WITH postdata AS (
   SELECT CAST(content::json->>'docs' AS json) AS docs
   FROM http_post('http://192.168.3.25:5984/articles/_find', '{"selector":{"title":{"$regex": "FreeBSD"}},"limit": 10}', 'application/json')
 ),
 resultdocs AS (
   SELECT json_array_elements(docs) AS doc FROM postdata
 )
SELECT doc->>'_id' AS id, doc->>'title' AS title FROM resultdocs;
```
```
                    id                    |                                         title                                         
------------------------------------------+---------------------------------------------------------------------------------------
 007c351df62d0abb42810ee1b8be16ada89b67d2 | OpenLIBM: Replace nextafterl function with FreeBSD's version
 00f26b4d7abba1599972498e50ccdb2c2afd33a9 | FreeBSD 11.0 RC3 Released, OS Still Trying To Get Out This Month
 011b7c7e6c8c0d189f3d48beced1fa8e99689d76 | FreeBSD 11.3 Beta 2 Brings Virtualization Updates, Exposes MD_CLEAR MDS Bit To Guests
 0132a0b6f09d9a7dd48af6a6262578e030d3a217 | sbin/newfs_msdos: Bring in FreeBSD/Git 47266d91, b25a2bc0
 013924c18386b65224082db1b58caabbfdaf11e9 | Bring in efibootmgr(8) from FreeBSD.
 013999e30ae9e261f468096b6e568461754f92bf | bsd-family-tree: Sync with FreeBSD (DragonFly 4.6.0).
 0169ecc2b421d2538cbbf478d0643b8061c3faf2 | Getting RabbitMQ running: FreeBSD 9.2
 017ad2ca04ce90602ed97ca95e189c4c149aebe9 | makewhatis(8): small cleanups, reduce diff against FreeBSD
 01db4f923c378c86e4f7b37808f7d51b37a2b035 | bsd-family-tree: Sync with FreeBSD (2.11BSD release date fix).
 022908a02b83aa81b32a3572f44af97a3a5e6c2d | FreeBSD 9.2 Is Behind Schedule, RC4 Released
(10 rows)

Time: 225.286 ms

```
Its useful to check on stuff with curl on the command line sometimes below is example of the POST request:

```curl -H 'Content-Type: application/json' -X POST http://192.168.3.25:5984/articles/_find -d '{"selector":{"title":{"$regex": "FreeBSD"}},"limit": 10}' -u admin:pass -i```


Its possible to add/update the data in couchdb using this extension. For example the below trigger can be added to a table or view to also update couch when postgres data is updated.

```
CREATE OR REPLACE FUNCTION couchdb_post() RETURNS trigger AS $BODY$
  DECLARE
      RES RECORD;
  BEGIN
   IF (NEW.from_pg) IS NULL THEN
     RETURN NEW;
   ELSE 
     
     SELECT status FROM http_post('http://192.168.3.21:5984/' || TG_TABLE_NAME || '/' || NEW.id::text, '', NEW.doc::text, 'application/json'::text) INTO RES;    
     --Need to check RES for response code
     --RAISE EXCEPTION 'Result: %', RES;
     RETURN null;
   END IF;
  END;
  $BODY$
LANGUAGE plpgsql VOLATILE  
```
 

## Elasticsearch 

Assuming ES already has the data from the previous couchdb example, later on will give example how this was done.

```curl -XGET 'http://192.168.3.20:9200/_cat/indices?v&pretty' -q```


Curl query:

curl -X GET "http://192.168.3.20:9200/myindex/_search?pretty" -H 'Content-Type: application/json' -d'{"query": {"query_string": {"query": "(FreeBSD) OR (Postgres)","fields": ["title","description"]}}}'


TODO - SQL example of this query to show how to get ES data into postgres.



### Example of loading data to ES

Sticking with the key, doc simple example with the doc being a json document.

```
CREATE TABLE public.estable (
    key text,
    doc jsonb
);
```


We need a trigger function to update ES when table is updated.

```
CREATE OR REPLACE FUNCTION upsert_es() RETURNS trigger AS $BODY$
  DECLARE
      RES RECORD;
      ESDOC jsonb;  
  BEGIN
     
     ESDOC = NEW.doc #- '{"_id"}'; -- need to remove _id field from json doc or es gets upset
                                   -- note other processing can be done before data is set to es
                                   
     SELECT * FROM http_post('http://192.168.3.20:9200/myindex/_update/' || NEW.key, 
                                  '{"doc":' || ESDOC::text || ',  "doc_as_upsert": true}', 
                                  'application/json'::text) INTO RES;    
                                  
     --Need to check RES for response code
     --RAISE EXCEPTION 'Result: %', RES.content; -- enable for debugging
     RETURN null;
  END;
  $BODY$
LANGUAGE plpgsql VOLATILE;
```

Add add trigger to the table:

```
CREATE TRIGGER es_upsert_trigger AFTER INSERT OR UPDATE ON public.estable FOR EACH ROW EXECUTE FUNCTION public.upsert_es();
```

Lets grab the data we had from the couchdb materialized view and insert into the estable which the trigger in turn should insert into ES.

```
INSERT INTO estable (key, doc)
   SELECT key, doc FROM couchdata;
```


Below is output from pgsql as you can see it takes over 3 hours to insert into ES as we are submitting each record separatley.

```
# INSERT INTO estable (key, doc) SELECT key, doc FROM couchdata;
INSERT 0 295417
Time: 13020441.452 ms (03:37:00.441)
```

A quick test to see how fast this is when records already exist in ES is its just a noop

```
# delete from estable;   
DELETE 295417
Time: 16954.411 ms (00:16.954)
# INSERT INTO estable (key, doc)
   SELECT key, doc FROM couchdata;
INSERT 0 295417
Time: 453780.787 ms (07:33.781)
```

So a significant improvement.


Lets bulk insert the records into ES 

```
WITH alldocs AS ( 
  SELECT key, doc #- '{"_id"}' AS doc 
  FROM couchdata ORDER BY key
),
alldocswithextra AS ( 
  SELECT key, '{ "index":{"_id" : "' || key || '"} }' AS doc, '1' AS myorder FROM alldocs 
  UNION ALL 
  SELECT key, doc::text, '2' AS myorder FROM alldocs ORDER BY key, myorder
),
chunked AS ( 
  SELECT doc, ((ROW_NUMBER() OVER (ORDER BY key) - 1)  / 1000) +1 AS chunk_no  --make batch count divisible by 2
  FROM alldocswithextra
),
chunked_docs AS (  -- Bulk up bulk_docs chunks to send 
    SELECT array_agg(doc) AS bulk_docs, chunk_no FROM chunked GROUP BY chunk_no  ORDER BY chunk_no
)
SELECT chunk_no, status 
FROM chunked_docs,
     http_post('http://192.168.3.20:9200/secondindex/_bulk', 
     array_to_string(array_append(bulk_docs,E'\n'),E'\n'),
     'application/x-ndjson'::text); 
```

250 batch size:
Time: 352147.458 ms (05:52.147)

500 batch size:
Time: 231784.233 ms (03:51.784)

1000 batch size:
Time: 238769.165 ms (03:58.769)

So under 4 minutes which is only about 20 secs slower than when there was already a document in ES.

The bulk update to ES could be in a function to be called whenever a full index is required and the trigger function can then take care of any updates/inserts.

TODO: Need to add DELETE example trigger.



## Pagination
How to deal with API which have limited number of items per page and do not allow you to access them in one request.
Create function to load each page/chunk to a temp table.
Another way maybe to run UNION ALL with multpile requests to server with page_no incrementing

## Other uses
Mailgun or twillio to download and archive logs as they limited retention period but logs are exposed via api.
Other useful APIs to example use with?




