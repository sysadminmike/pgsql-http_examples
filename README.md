# pgsql-http_examples
Example uses for PostgreSQL HTTP Client - https://github.com/pramsey/pgsql-http 

There is little infomation about this extension as I do not think people realise how useful it is, hopefully these examples will give other people ideas of how useful it can be.


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
 Algeria                                      | DZ
 ...
 Yemen                                        | YE
 Zambia                                       | ZM
 Zimbabwe                                     | ZW
(249 rows)
```

Much more useful, as with all examples queries could include joins to other tables and views in a database.

