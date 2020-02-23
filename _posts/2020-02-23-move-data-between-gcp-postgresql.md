---
layout: post
title:  "Move data between PostgreSQL instances on GCP"
date:   2020-02-23 6:00:00 +0000
categories: [postgresql, gcp]
tags: [migration, pg_dump, psql, docker]
---

A while ago I had to migrate the data from one database to the other on GCP. You can try using export/import functionality described [here](https://cloud.google.com/sql/docs/postgres/import-export/importing)

If you have to move data between different clouds or you want to copy data to a database on a localhost, you may want to use pg_dump and psql.

## Running databases with Docker locally

If you want to test it on localhost, you can use docker to run postgres. Let’s run two databases, we create database called db_source where we will have our data and other called db_destination where we want to replicate what is in source. While I'm using databases running locally, you can connect to a database running in a cloud. I like to play first with my environment, it makes me comfortable with what I'm doing, before I try it on a cloud instance and something breaks.

Run source database.
{% highlight shell %}
docker run --rm \
--name db_source \
-e POSTGRES_USER=username \
-e POSTGRES_PASSWORD=pass \
-e POSTGRES_DB=db_source \
-d -p 5433:5432 \
postgres:latest
{% endhighlight %}

Run destination database.
{% highlight shell %}
docker run --rm \
--name db_destination \
-e POSTGRES_USER=username2 \
-e POSTGRES_PASSWORD=pass2 \
-e POSTGRES_DB=db_destination \
-d -p 5434:5432 \
postgres:latest
{% endhighlight %}

When you run `docker ps` you should see information about your databases.

```
CONTAINER ID  IMAGE            COMMAND                 CREATED             STATUS             PORTS                   NAMES
da36c2674992  postgres:latest  "docker-entrypoint.s…"  4 seconds ago       Up 2 seconds       0.0.0.0:5434->5432/tcp  db_destination
ae5760af4329  postgres:latest  "docker-entrypoint.s…"  About a minute ago  Up About a minute  0.0.0.0:5433->5432/tcp  db_source
```

## Create test data

Let’s create some data in db_source.

{% highlight shell %}
psql \
-h 0.0.0.0 \
-p 5433 \
-U username \
-d db_source \
-c "CREATE TABLE example(value INT);
INSERT INTO example(value)
VALUES (1),(2),(3),(4);"
{% endhighlight %}

You will be asked for a password, for db_source we set the password as `pass` with a POSTGRES_PASSWORD environment variable.
You could pass (do you see what I did here?) it as an environment variable to psql, but it’s not recommended as it would stay in your command history. For this demo we already have password in a history, because we set it when creating a docker container, so you may carry on with this approach. I will show you what is a faster and safer way later.
It would look like that.

{% highlight shell %}
PGPASSWORD=pass \
psql \
-h 0.0.0.0 \
-p 5433 \
-U username \
-d db_source \
-c "SELECT * FROM example;"
{% endhighlight %}

If you want to stay connected to database to write more queries you can run psql without any commands.

{% highlight shell %}
psql \
-h 0.0.0.0 \
-p 5433 \
-U username \
-d db_source
{% endhighlight %}

Once you connect run `\dt` to see your table.

## Generate table backup

Now we would like to copy data from source to destination. Let’s first see what pg_dump will generate. It will ask you for a password again.

{% highlight shell %}
pg_dump \
-h 0.0.0.0 \
-p 5433 \
-U username \
-t example \
db_source
{% endhighlight %}

You can see, as part of the output, table and data we inserted.

```
CREATE TABLE public.example (
    value integer
);


ALTER TABLE public.example OWNER TO username;

--
-- Data for Name: example; Type: TABLE DATA; Schema: public; Owner: username
--

COPY public.example (value) FROM stdin;
1
2
3
4
\.
```

## No more typing passwords

Typing password every time is annoying, let’s create a file with passwords for our databases, it’s called .pgpass (in home directory so ~/.pgpass) and it’s format is host:port:dbname:username:password
For our databases it will look like that.

```
0.0.0.0:5433:db_source:username:pass
0.0.0.0:5434:db_destination:username2:pass2
```

For this file to be used you have to change permission to 0600.

{% highlight shell %}
chmod 0600 ~/.pgpass
{% endhighlight %}

## Restore in destination database

Now let's go back to our main quest, without typing passwords all the time, you can redirect output to a file.

{% highlight shell %}
pg_dump \
-h 0.0.0.0 \
-p 5433 \
-U username \
-t example db_source \
> migration.sql
{% endhighlight %}

Now we have a file and we can import it to our destination database.

{% highlight shell %}
psql \
-h 0.0.0.0 \
-p 5434 \
-U username2 \
-d db_destination \
-f migration.sql
{% endhighlight %}

Now you can query your database to see if table was migrated correctly.

{% highlight shell %}
psql \
-h 0.0.0.0 \
-p 5434 \
-U username2 \
-d db_destination \
-c "SELECT * FROM example;"
{% endhighlight %}

That's all good, but instead of creating a file you can redirect pg_dump output to psql, let's first delete existing table.

{% highlight shell %}
psql \
-h 0.0.0.0 \
-p 5434 \
-U username2 \
-d db_destination \
-c "DROP TABLE example;"
{% endhighlight %}

We can run migration again.
{% highlight shell %}
pg_dump \
-U username \
-h 0.0.0.0 \
-p 5433 \
-t example \
db_source |\
psql \
-h 0.0.0.0 \
-p 5434 \
-U username2 \
-d db_destination
{% endhighlight %}

Check if you have data is in destination database.

## When it doesn't work with creating an index after

I had to run it for a table which had size around 36GB and 220M rows. Generated dump first creates a table, then inserts all the data and after that creates constraints and indices. You can see it by creating new table.

{% highlight shell %}
psql \
-h 0.0.0.0 \
-p 5433 \
-U username \
-d db_source \
-c "CREATE TABLE example2(value INT UNIQUE);
INSERT INTO example2(value)
VALUES (1),(2),(3),(4);"
{% endhighlight %}

When you run pg_dump.

{% highlight shell %}
pg_dump \
-U username \
-h 0.0.0.0 \
-p 5433 \
-t example2 \
db_source
{% endhighlight %}

You can see output containing:

```
CREATE TABLE public.example2 (
    value integer
);


ALTER TABLE public.example2 OWNER TO username;

--
-- Data for Name: example2; Type: TABLE DATA; Schema: public; Owner: username
--

COPY public.example2 (value) FROM stdin;
1
2
3
4
\.


--
-- Name: example2 example2_value_key; Type: CONSTRAINT; Schema: public; Owner: username
--

ALTER TABLE ONLY public.example2
    ADD CONSTRAINT example2_value_key UNIQUE (value);
```

It’s more efficient to create a table, insert data without any checks, and once we have all rows we can create indices. It should be faster operation than validating every row when inserting data. Because of the size of table and limits on my PostgreSQL instance, I was getting an exception when creating an index. My work around was to migrate the schema first and then data. To do that, you have to use `--schema-only` and `--data-only`, see example below.

{% highlight shell %}
pg_dump \
-U username \
-h 0.0.0.0 \
-p 5433 \
-t example2 \
db_source \
--schema-only |\
psql \
-h 0.0.0.0 \
-p 5434 \
-U username2 \
-d db_destination
{% endhighlight %}

{% highlight shell %}
pg_dump \
-U username \
-h 0.0.0.0 \
-p 5433 \
-t example2 \
db_source \
--data-only |\
psql \
-h 0.0.0.0 \
-p 5434 \
-U username2 \
-d db_destination
{% endhighlight %}

You can check output from both dumps when you remove psql part from the query, like we did at the beginning when we displayed output on a console.

I hope you found it useful, good luck on your data migration!
