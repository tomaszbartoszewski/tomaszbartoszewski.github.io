---
layout: post
title:  "Running pgTAP in Docker with Flyway schema definition"
date:   2020-04-08 10:00:00 +0000
categories: [postgresql, pgtap]
tags: [tests, flyway, docker, docker-compose]
---

In this post I will show you how I'm testing functions in PostgreSQL using pgTAP. PostgreSQL and Flyway are running with Docker Compose, while pgTAP is running in separated container.

## What I will cover here

In this post I will show you how to run pgTAP using Docker. In my case I wanted to run it after Flyway applies a schema. While I will show you how to run a test, I won't go here into details how pgTAP works. You can use my example to get something running locally, but if you are just getting started, you may prefer to run it without Docker first. On the other hand my example is a good playground.

## Credits

I used a lot of work done by other people. Big parts of it I copied and then adjusted to my needs.

This was the approach I used first [docker-pgtap](https://github.com/walm/docker-pgtap) it worked for older version of PostgreSQL, but failed for 12.2. After trying to install dependencies for plv8, image was getting massive, so I kept the script but changed Dockerfile based on this [comment](https://github.com/docker-library/postgres/issues/306#issuecomment-345063809)

Blog posts which I found very useful:
[Unit testing with pgTAP](https://medium.com/engineering-on-the-incline/unit-testing-postgres-with-pgtap-af09ec42795) &
[Testing functions](https://medium.com/engineering-on-the-incline/unit-testing-functions-in-postgresql-with-pgtap-in-5-simple-steps-beef933d02d3)

And as usual [documentation](https://pgtap.org/) was helpful.

## Requirements

* Docker
* Docker Compose
* [Code from GitHub](https://github.com/tomaszbartoszewski/postgresql-docker)
* Psql (not essential but is useful)
* [Understanding of existing Flyway migration]({% post_url 2020-03-15-running-postgresql-and-flyway-with-docker-compose %}) (optional)

## What is inside Docker image

To run PostgreSQL and Flyway from my repository, you have to run
{% highlight shell %}
docker-compose -f docker-compose-postgres.yml up
{% endhighlight %}

That will start PostgreSQL, then Flyway will apply schema from `sql_versions` directory. To run tests we have to wait until PostgreSQL is ready. It will accept connection when you can call it's endpoint and get empty reply from the server.
{% highlight shell %}
curl http://127.0.0.1:5432
{% endhighlight %}
It should give you a response:
```
curl: (52) Empty reply from server
```
This doesn't mean that Flyway finished running, when you have many migration scripts, it can take a while. For humans it will be quick, but when you have a script which tries to run tests, migration is too slow.

Our next step is to wait for Flyway. I achieved that by querying PostgreSQL and making sure that Flyway applied all files from `sql_versions` directory. This is part of other script, but it can help with understanding.
{% highlight shell %}
COUNT_FILES_TO_EXECUTE=$(ls -1 sql_versions | wc -l)
{% endhighlight %}
Because some scripts can be applied multiple times, starting by default with `R__`, I had to use `count(DISTINCT script)` to get correct number, if we run on empty database `count(*)` would work too.

Try to run command below, I had to remove spaces from response to get just a number.
While for our tests we can pass password with environment variable, be careful to not do the same with your production database, as it will stay in your command history.
{% highlight shell %}
PGPASSWORD=pass psql -h 127.0.0.1 -p 5432 -U example-username \
-d db-name -t -c \
'SELECT count(DISTINCT script) FROM flyway_schema_history;' |\
tr -d '[:space:]'
{% endhighlight %}
Without changing schema files, you should see `5` as the only output.
Because you can run this query in the middle of Flyway migration, I had to compare this number with number of files to apply. This part of bash script does it.
{% highlight bash %}
APPLIED_SCRIPTS=0
echo "$FLYWAY_SCRIPTS_COUNT scripts to apply"
while [ "$APPLIED_SCRIPTS" -ne "$FLYWAY_SCRIPTS_COUNT" ]
do
    echo "Waiting for Flyway to finish"
    APPLIED_SCRIPTS_RESPONSE=$(PGPASSWORD=$PASSWORD psql -h $HOST -p $PORT \
    -U $USER -d $DATABASE -t -c \
    'SELECT count(DISTINCT script) FROM flyway_schema_history;' |\
    tr -d '[:space:]')
    if [ ! -z "$APPLIED_SCRIPTS_RESPONSE" ]; then
        APPLIED_SCRIPTS=$APPLIED_SCRIPTS_RESPONSE
        echo "$APPLIED_SCRIPTS scripts executed already"
    fi
    sleep 1
done
{% endhighlight %}

Once it's done we can install pgTAP inside PostgreSQL. I'm using sql file, which I have inside pgTAP Docker container.
{% highlight bash %}
PGPASSWORD=$PASSWORD psql -h $HOST -p $PORT \
-d $DATABASE -U $USER \
-f /usr/local/share/postgresql/extension/pgtap.sql > /dev/null
{% endhighlight %}

The last step is to execute tests which happens with
{% highlight bash %}
PGPASSWORD=$PASSWORD pg_prove -h $HOST -p $PORT -d $DATABASE -U $USER $TESTS
{% endhighlight %}

Most of code above you can find in this [script](https://github.com/tomaszbartoszewski/postgresql-docker/blob/master/tests_docker/test.sh), it is part of generated Docker image. This way my co-workers can pull an image from Container Registry and use it quickly.

You can build Docker image with make command.
{% highlight shell %}
make build_test_image
{% endhighlight %}

## Running tests

After creating Docker image we can run tests. We have to start PostgreSQL with Flyway in a background
{% highlight shell %}
docker-compose -f docker-compose-postgres.yml up > /dev/null &
{% endhighlight %}

The next step is to start Docker container with pgTAP from previous section. We have to mount directory with our tests and pass all parameters required by `test.sh`, which is an entry point in our container.
{% highlight bash %}
COUNT_FILES_TO_EXECUTE=$(ls -1 sql_versions | wc -l)
docker run --rm --network="host" \
-v $(pwd)/tests:/tests pgtap-tester:latest \
-h 127.0.0.1 -p 5432 -u example-username -w pass -d db-name \
-t '/tests/*.sql' -f $COUNT_FILES_TO_EXECUTE  2>&1 | tee test_output.txt
{% endhighlight %}

Then we want to know if our tests were successful, I search `test_output.txt` for this information.
{% highlight bash %}
TEST_RESULT=$(cat test_output.txt | grep Result: | cut -c9-100)
{% endhighlight %}

Then I used `cat` to print test output so we can see more details.

You can check full script [here](https://github.com/tomaszbartoszewski/postgresql-docker/blob/master/tests_runner.sh)

## What tests are running.

You can see there is one [test file](https://github.com/tomaszbartoszewski/postgresql-docker/blob/master/tests/test_save_car_and_person.sql), you can add more sql files to this directory. It's running two tests, one without any data in tables and second test where there is some existing data. You can see function definition [here](https://github.com/tomaszbartoszewski/postgresql-docker/blob/master/sql_versions/R__Save_car_and_person.sql).

If you want to run parts of test manually, you can do it with psql. After running database in Docker Compose, you can connect to it.
{% highlight shell %}
PGPASSWORD=pass psql -h 127.0.0.1 -p 5432 -U example-username -d db-name
{% endhighlight %}

Inside psql you can create variables.
```
\set first_name '\'Audrey\''
\set last_name '\'Jones\''
\set car_make '\'Mitsubishi\''
\set car_model '\'Outlander PHEV\''
```

Then you can use them to call our function.
{% highlight sql %}
SELECT save_car_and_person(:first_name::varchar(50),
                           :last_name::varchar(50),
                           :car_make::varchar(50),
                           :car_model::varchar(50));
{% endhighlight %}

The next step is to query tables.
{% highlight sql %}
PREPARE observed_values AS
    SELECT
        p.first_name,
        p.last_name,
        c.make,
        c.model
    FROM person p
    INNER JOIN person_car pc ON pc.person_id = p.id
    INNER JOIN car c ON c.id = pc.car_id
    WHERE p.first_name = :first_name AND p.last_name = :last_name;
{% endhighlight %}

So far all of it is inside a test file. If you want to see results you will have to run execute.
{% highlight sql %}
EXECUTE observed_values;
{% endhighlight %}

That should give you enough information if your test code is working. If you would prefer to run a full test manually, including pgTAP functions, you have to remove one line from `tests_runner.sh` which stops and removes database:
{% highlight shell %}
docker-compose -f docker-compose-postgres.yml down -v
{% endhighlight %}
Without it, after you run your tests, database will stay open with pgTAP installed. You can copy and paste entire test into psql, if you remove `ROLLBACK`, you can query data left in a database after a test. It is helpful for debugging tests, as you can run one command at a time and check if data is as expected.

## Happy testing
While writing tests for PostgreSQL with pgTAP is not an easy process, it is extremely useful. Being able to run it in Docker not only allow you to run it locally, but gives your co-workers easy start, without lots of installation steps. If your build server supports running Docker images, making it part of CI will give you an extra confidence in your database changes, before they hit any environment. Go and add some tests!