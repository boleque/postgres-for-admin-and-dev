## Table Sectioning

### 1. Hash

```
-- Create a separate schema
CREATE SCHEMA IF NOT EXISTS by_hash;

-- Create the table
CREATE TABLE by_hash.flights_child (
	flight_id serial4 NOT NULL,
	flight_no bpchar(6) NOT NULL,
	scheduled_departure timestamptz NOT NULL,
	scheduled_arrival timestamptz NOT NULL,
	departure_airport bpchar(3) NOT NULL,
	arrival_airport bpchar(3) NOT NULL,
	status varchar(20) NOT NULL,
	aircraft_code bpchar(3) NOT NULL,
	actual_departure timestamptz NULL,
	actual_arrival timestamptz NULL
) PARTITION BY hash (flight_id);

-- Create partition tables, 5 in total
CREATE TABLE by_hash.flights_child_0 partition of by_hash.flights_child for values with (modulus 5, remainder 0);
CREATE TABLE by_hash.flights_child_1 partition of by_hash.flights_child for values with (modulus 5, remainder 1);
CREATE TABLE by_hash.flights_child_2 partition of by_hash.flights_child for values with (modulus 5, remainder 2);
CREATE TABLE by_hash.flights_child_3 partition of by_hash.flights_child for values with (modulus 5, remainder 3);
CREATE TABLE by_hash.flights_child_4 partition of by_hash.flights_child for values with (modulus 5, remainder 4);

-- Populate the child table
INSERT INTO by_hash.flights_child
SELECT * FROM bookings.flights;

-- Analyze query plans
-- 1. Query on the parent table
demo=# explain select * from bookings.flights where flight_id = 100;
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Index Scan using flights_pkey on flights  (cost=0.42..8.44 rows=1 width=63)
   Index Cond: (flight_id = 100)
(2 rows)

-- 2. Query on the child table - planner directs us to partition #4, i.e., table flights_child_4, everything is correct
demo=# explain select * from by_hash.flights_child where flight_id = 100;
                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Seq Scan on flights_child_4 flights_child  (cost=0.00..1063.94 rows=1 width=63)
   Filter: (flight_id = 100)
(2 rows)

```

### 1. List

```
-- Create a separate schema
CREATE SCHEMA IF NOT EXISTS by_list;

-- Create the table
CREATE TABLE by_list.flights_child (
	flight_id serial4 NOT NULL,
	flight_no bpchar(6) NOT NULL,
	scheduled_departure timestamptz NOT NULL,
	scheduled_arrival timestamptz NOT NULL,
	departure_airport bpchar(3) NOT NULL,
	arrival_airport bpchar(3) NOT NULL,
	status varchar(20) NOT NULL,
	aircraft_code bpchar(3) NOT NULL,
	actual_departure timestamptz NULL,
	actual_arrival timestamptz NULL
) PARTITION BY list (departure_airport);

-- Create partition tables, 3 in total - departure airports from Moscow, St. Petersburg, and others
CREATE TABLE by_list.departure_airport_msk PARTITION OF by_list.flights_child
    FOR VALUES IN ('VKO', 'SVO');

CREATE TABLE by_list.departure_airport_spb PARTITION OF by_list.flights_child
    FOR VALUES IN ('LED');

CREATE TABLE by_list.departure_airport_others PARTITION OF by_list.flights_child DEFAULT;

-- Populate the child table
INSERT INTO by_list.flights_child
SELECT * FROM bookings.flights;

-- Analyze query plans
-- 1. Query for departure_airport = VKO directs us to the correct partition
demo=# EXPLAIN SELECT * FROM by_list.flights_child WHERE departure_airport = 'VKO';
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Seq Scan on departure_airport_msk flights_child  (cost=0.00..754.15 rows=11138 width=63)
   Filter: (departure_airport = 'VKO'::bpchar)
(2 rows)

-- 2. Query for departure_airport = KZN directs us to the default partition, as expected the query cost increases
demo=# EXPLAIN SELECT * FROM by_list.flights_child WHERE departure_airport = 'KZN';
                                         QUERY PLAN                                          
---------------------------------------------------------------------------------------------
 Seq Scan on departure_airport_others flights_child  (cost=0.00..4251.54 rows=3137 width=63)
   Filter: (departure_airport = 'KZN'::bpchar)
(2 rows)

```