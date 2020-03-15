---
layout: post
title:  "Running PostgreSQL and Flyway with Docker Compose"
date:   2020-03-15 15:00:00 +0000
categories: [postgresql, flyway]
tags: [migration, flyway, docker, docker-compose]
---

Run PostgreSQL locally and apply schema changes using Flyway. Let's see, how to do it without messing up with your local environment, thanks to Docker Compose.

## Use case

If you ever worked with relational databases, you know that migrating schema is not trivial. While deploying your application in most cases you just have to replace your code. With databases deployments, making sure data is correct, is not so easy. Fortunately there are many tools which can make it simpler. I'm using Flyway at work and I like to run my services locally, so today I will show you how you can do that.

## Requirements

If you want to run this example locally, you will need:
* Docker
* Docker Compose
* Psql or any other database client which can connect to PostgreSQL
* [Code from GitHub](https://github.com/tomaszbartoszewski/postgresql-docker)

## Essential knowledge about Flyway

[Flyway](https://flywaydb.org/) is a tool which keeps track of your last applied schema and performs migration to latest version. Community Edition is free and so far worked well for me. In this example I'm defining database changes with SQL files.
What you should know about Flyway, it allows you to run versioned changes. It keeps information in a database about changes history. By default files starting with V are versioned changes, they run once and any change to a file will result in blocked migration.
Second type of files are repeatable migrations, they will run every time the checksum changes. You have to make sure they can run on top of the previous migration.
Order of execution is versioned then repeatable changes.
More about it you can find in the [Flyway documentation](https://flywaydb.org/documentation/migrations)

## Migration definition

If you look at files in [sql_versions](https://github.com/tomaszbartoszewski/postgresql-docker/tree/master/sql_versions) directory, you will see `V1__Create_person_table.sql` which creates one table.

{% highlight sql %}
CREATE TABLE person (
    id serial PRIMARY KEY,
    first_name varchar(50),
    last_name varchar(50)
);
{% endhighlight %}

Your updates can create more, if you have a look at `V2__Create_car_and_person_car_tables.sql` you will notice it creates two tables.

{% highlight sql %}
CREATE TABLE car (
    id serial PRIMARY KEY,
    make varchar(50),
    model varchar(50)
);

CREATE TABLE person_car (
    person_id integer
    CONSTRAINT person_car_person_id__fk
    REFERENCES person(id),
    car_id integer
    CONSTRAINT person_car_car_id__fk
    REFERENCES car(id)
);
{% endhighlight %}

They will execute in ascending order. After all versioned files, Flyway will run file `R__person_car_detail_view.sql` which creates a view.

{% highlight sql %}
CREATE OR REPLACE VIEW person_car_detail AS
SELECT p.first_name,
       p.last_name,
       c.make AS car_make,
       c.model AS car_model
FROM person p
INNER JOIN person_car pc ON pc.person_id = p.id
INNER JOIN car c ON c.id = pc.car_id;
{% endhighlight %}

All of the above queries, you could run manually on an empty database, as it's all SQL. I may add some files for other posts, but it should not affect this example.

## Running Flyway

You could run Flyway directly on your machine, but that requires everyone to install Flyway. If your team is using Docker Compose already, it's just easier to do it this way.

You need two things. Flyway configuration with database details and Docker Compose yaml file.

Let's look first at Docker Compose as that's where you define a database.

{% highlight yaml %}
version: '3'
services:
  flyway:
    image: flyway/flyway:6.3.1
    command: -configFiles=/flyway/conf/flyway.config -locations=filesystem:/flyway/sql -connectRetries=60 migrate
    volumes:
      - ${PWD}/sql_versions:/flyway/sql
      - ${PWD}/docker-flyway.config:/flyway/conf/flyway.config
    depends_on:
      - postgres
  postgres:
    image: postgres:12.2
    restart: always
    ports:
    - "5432:5432"
    environment:
    - POSTGRES_USER=example-username
    - POSTGRES_PASSWORD=pass
    - POSTGRES_DB=db-name
{% endhighlight %}

You can see we create two services, postgres and flyway. Postgres is using Docker image `postgres:12.2`. We are using the default PostgreSQL port, but if you have something running already on 5432, you could change mapping to something like `"5433:5432"`, it will make your instance accessible on port 5433.
Environment variables provide information about database instance name, user name and password. We will need all three for Flyway configuration.
Second service is Flyway, we are using Docker image `flyway/flyway:6.3.1`. While Docker Compose allow you to specify starting order by using `depends_on`, it doesn't work with PostgreSQL container. We start a database but it takes a while for it to accept connections. That's why in command you can see a flag `-connectRetries=60`. Before we look at other command parameters, let's check volumes. We have to provide SQL migration scripts and configuration. Format for volumes is `location on disc`:`mounted as`. When we have volumes, we can use them in our command, look at `-configFiles=/flyway/conf/flyway.config` and `-locations=filesystem:/flyway/sql`. Final bit is to say what Flyway has to do, in this case it's `migrate`.

Let's have a look at a Flyway configuration now.
```
flyway.url=jdbc:postgresql://postgres:5432/db-name
flyway.user=example-username
flyway.password=pass
flyway.baselineOnMigrate=false
```
We have to specify database url, it contains a jdbc driver, service name, we called it in Docker Compose yaml - postgres. We need port and then instance name, defined in POSTGRES_DB.
Two other parameters are credentials - user name and password, check POSTGRES_USER and POSTGRES_PASSWORD.
Final setting is `baselineOnMigrate` which tells Flyway, if it's fine to run on non empty database. If you create brand new database, as we do in this example, we set it to false.

## Let's run it
We have everything set up, it's time to run our database.

{% highlight shell %}
docker-compose -f docker-compose-postgres.yml up
{% endhighlight %}

You should see in the output information about refused connection `flyway_1    | WARNING: Connection error: Connection to postgres:5432 refused. (...) Retrying in 1 sec...`, this is why we had to set `connectRetries`. Couple lines later database is ready to accept connections `postgres_1  | 2020-03-15 19:05:42.578 UTC [1] LOG:  database system is ready to accept connections`. After that, Flyway will apply schema changes.
```
flyway_1    | Database: jdbc:postgresql://postgres:5432/db-name (PostgreSQL 12.2)
flyway_1    | Successfully validated 3 migrations (execution time 00:00.057s)
flyway_1    | Creating Schema History table "public"."flyway_schema_history" ...
flyway_1    | Current version of schema "public": << Empty Schema >>
flyway_1    | Migrating schema "public" to version 1 - Create person table
flyway_1    | Migrating schema "public" to version 2 - Create car and person car tables
flyway_1    | Migrating schema "public" with repeatable migration person car detail view
flyway_1    | Successfully applied 3 migrations to schema "public" (execution time 00:00.409s)
postgresql-docker_flyway_1 exited with code 0
```

Our database is ready.

## Connect to the database
I'm using Psql to check the database schema. It will ask you to type in a password, we set it earlier to `pass`.

{% highlight shell %}
psql \
-h 0.0.0.0 \
-p 5432 \
-U example-username \
-d db-name
{% endhighlight %}

When you connect you can list all tables in public schema.
```
\dt
```

It should give you:
```
                     List of relations
 Schema |         Name          | Type  |      Owner       
--------+-----------------------+-------+------------------
 public | car                   | table | example-username
 public | flyway_schema_history | table | example-username
 public | person                | table | example-username
 public | person_car            | table | example-username
```

If you want to see views run:
```
\dv
```
Output:
```
                  List of relations
 Schema |       Name        | Type |      Owner       
--------+-------------------+------+------------------
 public | person_car_detail | view | example-username
```

If you have a look at `flyway_schema_history`, you should see rows matching files we had in `sql_versions` directory.

{% highlight sql %}
SELECT * FROM flyway_schema_history;
{% endhighlight %}

```
 installed_rank | version |           description            | type |                  script                  |  checksum  |   installed_by   |        installed_on        | execution_time | success 
----------------+---------+----------------------------------+------+------------------------------------------+------------+------------------+----------------------------+----------------+---------
              1 | 1       | Create person table              | SQL  | V1__Create_person_table.sql              | 1996238249 | example-username | 2020-03-15 19:05:43.462699 |             13 | t
              2 | 2       | Create car and person car tables | SQL  | V2__Create_car_and_person_car_tables.sql | 1812600351 | example-username | 2020-03-15 19:05:43.605999 |             17 | t
              3 |         | person car detail view           | SQL  | R__person_car_detail_view.sql            | -430177297 | example-username | 2020-03-15 19:05:43.732313 |              4 | t
```

Now for a last check, let's create some data in our tables and see if view gives them back correctly.

{% highlight sql %}
INSERT INTO person(id, first_name, last_name)
VALUES (1, 'Jack', 'Smith'),
       (2, 'Audrey', 'Jones');

INSERT INTO car(id, make, model)
VALUES (1, 'Mitsubishi', 'Outlander PHEV'),
       (2, 'Nissan', 'Leaf');

INSERT INTO person_car(person_id, car_id)
VALUES (1, 2),
       (2, 1);

SELECT * FROM person_car_detail;
{% endhighlight %}

You should see this output:
```
 first_name | last_name |  car_make  |   car_model    
------------+-----------+------------+----------------
 Jack       | Smith     | Nissan     | Leaf
 Audrey     | Jones     | Mitsubishi | Outlander PHEV
```

## Keeping data
When you want to stop your database, you have to press `Ctrl+C`. That will leave database with data on your machine. If you run again:
{% highlight shell %}
docker-compose -f docker-compose-postgres.yml up
{% endhighlight %}

You should see that Flyway is not applying any changes `flyway_1    | Schema "public" is up to date. No migration necessary.`. When you check your tables, data will still be there. It's convenient, when you work on something for more than a day and want to turn off your computer in between. If you want to remove database completely, you will have to delete volume.
{% highlight shell %}
docker-compose -f docker-compose-postgres.yml down -v
{% endhighlight %}

## Summary
In this blog post we went through creating a schema with SQL scripts, which were then executed by Flyway. We configured PostgreSQL and Flyway to run in Docker Compose. Then we run few queries to make sure our database works as expected. Final part was about stopping PostgreSQL and deleting instance completely. I hope you found it useful.