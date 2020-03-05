#### SQL Knowledge Assignment:
given a users table of the form - id, group_id


create temp table users (id bigserial, group_id bigint);
insert into users (group_id) values (1), (1), (1), (2), (1), (3);

     

        In this table, sorted by ID, you must:

        1 select continuous groups by group_id taking into account 
        the specified order of records (there are 4 of them)
        2 count the number of records in each group
        3 calculate the minimum record ID in the group

#### Solution:

##### 1.Preparations

    ## Create users table
    CREATE TABLE users (id bigserial, group_id bigint);    
    
    ## Create temp table
    test=# CREATE TEMP TABLE users(id bigserial, group_id bigint);
    
    
    postgres=# CREATE TEMP TABLE users(id bigserial, group_id bigint);
    CREATE TABLE
    
    postgres=# \d
                   List of relations
      Schema   |     Name     |   Type   |  Owner  
    -----------+--------------+----------+----------
    pg_temp_2 | users        | table    | postgres
    pg_temp_2 | users_id_seq | sequence | postgres
    
    
    ## Insert values
    postgres=# INSERT INTO users(group_id) VALUES(1),(1),(1),(2),(1),(3);
    INSERT 0 6


##### 2. select continuous groups by group_id taking into account  the specified order of records (there are 4 of them)

2.1 Assign rows


Query:

    SELECT                    
       id,                 
       group_id,                    
       ROW_NUMBER () OVER (ORDER BY id)                    
    FROM                  
       users;
       
Terminal:

    postgres=# SELECT
    postgres-#    id,
    postgres-#    group_id,
    postgres-#    ROW_NUMBER () OVER (ORDER BY id)
    postgres-# FROM
    postgres-#    users;
    id | group_id | row_number
    ----+----------+------------
      1 |        1 |          1
      2 |        1 |          2
      3 |        1 |          3
      4 |        2 |          4
      5 |        1 |          5
      6 |        3 |          6
    (6 rows)       
          

2.2 Divide the window into subsets

Query:

    SELECT
       id,
       group_id,
       ROW_NUMBER () OVER (
          PARTITION BY group_id
          ORDER BY
             id
    ) as set_number
    FROM
       users;

Terminal: 

    postgres=#     SELECT
    postgres-#        id,
    postgres-#        group_id,
    postgres-#        ROW_NUMBER () OVER (
    postgres(#           PARTITION BY group_id
    postgres(#           ORDER BY
    postgres(#              id
    postgres(#     ) as set_number
    postgres-#     FROM
    postgres-#        users;
     id | group_id | set_number 
    ----+----------+------------
      1 |        1 |          1
      2 |        1 |          2
      3 |        1 |          3
      5 |        1 |          4
      4 |        2 |          1
      6 |        3 |          1
    (6 rows)




##### 3. Count the number of records in each group

3.1 Using subquery (faster)

Query:

    SELECT continious_groups.set_number,
           COUNT (continious_groups.set_number)       
    FROM  ( SELECT
               id,
               group_id,
               ROW_NUMBER () OVER (
                  PARTITION BY group_id
                  ORDER BY
                     id
            ) as set_number
            FROM
               users) AS continious_groups
    GROUP BY continious_groups.set_number;

In terminal:


    postgres=#     SELECT continious_groups.set_number,
    postgres-#            COUNT (continious_groups.set_number)       
    postgres-#     FROM  ( SELECT
    postgres(#                id,
    postgres(#                group_id,
    postgres(#                ROW_NUMBER () OVER (
    postgres(#                   PARTITION BY group_id
    postgres(#                   ORDER BY
    postgres(#                      id
    postgres(#             ) as set_number
    postgres(#             FROM
    postgres(#                users) AS continious_groups
    postgres-#     GROUP BY continious_groups.set_number;
     set_number | count 
    ------------+-------
              4 |     1
              1 |     3
              3 |     1
              2 |     1
    (4 rows)



3.2 Using WITH (this option will eat more ram then the first one with subquery)

Query:

    WITH continious_groups AS (
    SELECT
       id,
       group_id,
       ROW_NUMBER () OVER (
          PARTITION BY group_id
          ORDER BY
             id
    ) as set_number
    FROM
       users)
    SELECT continious_groups.set_number,
           COUNT (continious_groups.set_number)       
    FROM continious_groups
    GROUP BY continious_groups.set_number;


Terminal:

    postgres=#     WITH continious_groups AS (
    postgres(#     SELECT
    postgres(#        id,
    postgres(#        group_id,
    postgres(#        ROW_NUMBER () OVER (
    postgres(#           PARTITION BY group_id
    postgres(#           ORDER BY
    postgres(#              id
    postgres(#     ) as set_number
    postgres(#     FROM
    postgres(#        users)
    postgres-#     SELECT continious_groups.set_number,
    postgres-#            COUNT (continious_groups.set_number)       
    postgres-#     FROM continious_groups
    postgres-#     GROUP BY continious_groups.set_number;
     set_number | count 
    ------------+-------
              4 |     1
              1 |     3
              3 |     1
              2 |     1
    (4 rows)






##### 4. Calculate the minimum record ID in the group

4.1 Query (subquery):  

    SELECT continious_groups.set_number, 
           MIN(continious_groups.id) AS min_id,
           COUNT (continious_groups.set_number)       
    FROM  ( SELECT
               id,
               group_id,
               ROW_NUMBER () OVER (
                  PARTITION BY group_id
                  ORDER BY
                     id
            ) as set_number
            FROM
               users) AS continious_groups
    GROUP BY continious_groups.set_number;


Terminal:
    
    postgres=#     SELECT continious_groups.set_number, 
    postgres-#            MIN(continious_groups.id) AS min_id,
    postgres-#            COUNT (continious_groups.set_number)       
    postgres-#     FROM  ( SELECT
    postgres(#                id,
    postgres(#                group_id,
    postgres(#                ROW_NUMBER () OVER (
    postgres(#                   PARTITION BY group_id
    postgres(#                   ORDER BY
    postgres(#                      id
    postgres(#             ) as set_number
    postgres(#             FROM
    postgres(#                users) AS continious_groups
    postgres-#     GROUP BY continious_groups.set_number;
     set_number | min_id | count 
    ------------+--------+-------
              4 |      5 |     1
              1 |      1 |     3
              3 |      3 |     1
              2 |      2 |     1
    (4 rows)


Results from the first query for self-check:

     id | group_id | set_number 
    ----+----------+------------
      1 |        1 |          1
      2 |        1 |          2
      3 |        1 |          3
      5 |        1 |          4
      4 |        2 |          1
      6 |        3 |          1
    (6 rows)



4.2 Query (WITH statement)

Query:

    SELECT continious_groups.set_number, 
           MIN(continious_groups.id) AS min_id,
           COUNT (continious_groups.set_number)       
    FROM  ( SELECT
               id,
               group_id,
               ROW_NUMBER () OVER (
                  PARTITION BY group_id
                  ORDER BY
                     id
            ) as set_number
            FROM
               users) AS continious_groups
    GROUP BY continious_groups.set_number;


Terminal: 

    postgres=# SELECT continious_groups.set_number, 
    postgres-#        MIN(continious_groups.id) AS min_id,
    postgres-#        COUNT (continious_groups.set_number)       
    postgres-# FROM  ( SELECT
    postgres(#            id,
    postgres(#            group_id,
    postgres(#            ROW_NUMBER () OVER (
    postgres(#               PARTITION BY group_id
    postgres(#               ORDER BY
    postgres(#                  id
    postgres(#         ) as set_number
    postgres(#         FROM
    postgres(#            users) AS continious_groups
    postgres-# GROUP BY continious_groups.set_number;
     set_number | min_id | count 
    ------------+--------+-------
              4 |      5 |     1
              1 |      1 |     3
              3 |      3 |     1
              2 |      2 |     1
    (4 rows)


Results from the first query for self-check:

     id | group_id | set_number 
    ----+----------+------------
      1 |        1 |          1
      2 |        1 |          2
      3 |        1 |          3
      5 |        1 |          4
      4 |        2 |          1
      6 |        3 |          1
    (6 rows)



##### If we use goup_id as foreign key, then we should use bigserial in both cases (for postgresql 9.3).
It takes same 8 bytes but we don't use negative numbers for foreign keys.


    bigserial        8 bytes        large autoincrementing integer        1 to 9223372036854775807
    bigint        8 bytes        large-range integer        -9223372036854775808 to 9223372036854775807

I'm not sure we should use 


Links:

    https://www.postgresql.org/docs/9.3/datatype-numeric.html
    https://dba.stackexchange.com/questions/188093/why-is-postgres-cte-slower-than-subquery
    https://www.postgresqltutorial.com/postgresql-count-function/
    https://www.postgresqltutorial.com/postgresql-subquery/
    https://www.postgresqltutorial.com/postgresql-cte/
    http://sqlfiddle.com/#!17/c43ea/6
    https://www.postgresqltutorial.com/postgresql-row_number/
    https://www.postgresqltutorial.com/postgresql-count-function/
    https://www.postgresql.org/message-id/o5qd6k$2ni$2@blaine.gmane.org
    https://stackoverflow.com/questions/51879075/combining-count-and-rank-postgresql
    https://stackoverflow.com/questions/8193688/postgresql-running-count-of-rows-for-a-query-by-minute
    https://zaiste.net/row_number_in_postgresql/


