## Installing and Configuring PostgreSQL in a Docker Container

Launched a database container on an AWS EC2 instance:  
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Launched a separate client container and connected to pg-server:  
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres
```
Working with the database from the **pg-client**:  
```sql
-- Create a new database
CREATE DATABASE otus;

-- Connect to it
\c otus

-- Create and populate the persons table
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 

```
Checked that the data is in place:  
```sql
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```
Deleted the server container:  
```
sudo docker rm -f pg-server
```
Restarted the container:  
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Tried to connect to the database from the local machine:  
```
psql -p 5432 -U postgres -h ec2-13-49-66-130.eu-north-1.compue.amazonaws.com -d postgres -W
Password: 
psql: error: connection to server at "ec2-13-49-66-130.eu-north-1.compute.amazonaws.com" (13.49.66.130), port 5432 failed: fe_sendauth: no password supplied
```
Resolved the connection issue by:  
- Allowing inbound traffic on port 5432 for the AWS security group
- Modifying the IPv4 local connections in pg_hba.conf from 127.0.0.1/32 to 0.0.0.0/0

Successfully connected to the database from the local machine and confirmed that the data remained intact despite deleting the container:  
```sql
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```
