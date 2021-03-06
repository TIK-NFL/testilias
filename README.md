
# TestILIAS

TestILIAS is an open source, portable, scalable software testing kit for verifying the correctness of essential functions of the Test & Assessment module of ILIAS 5.

It's self-contained, rather easy to install on a single local machine (tested so far on Linux and macOS) or a server, and allows to run tests using a configurable number of parallel test users.

TestILIAS checks that:

* answers that are given in a test are saved reliably during the test (i.e. saved answers do not change upon revisiting the questions).
* answers are reliably saved and exported reliably  after the test (i.e. the xls export representation exactly mirrors the answers given during the test).
* the computed scores are correctly computed according to the question definition.

TestILIAS works by importing and analyzing a given test (i.e. the test contents are not hard coded) and then executing random test runs with a configurable number of robot participants in parallel. Currently the following question types are supported:

* Single Choice
* Multiple Choice
* KPrim
* Cloze Question (select, text, and numeric gaps)
* Long Text ("Essay") Question

TestILIAS allows to work around a number of known problems in order to perform test without already known fails. It also supports running tests in a loop mode so you can keep running randomized tests for a longer time.

# Getting Started

TestILIAS can be run on your local machine or on a server. The first option is fine for trying things out, for longer testing you'll want the second option though. You need to have <a href="https://www.docker.com/community-edition">docker-compose</a> and <a href="https://www.python.org/">python 2 or 3</a> installed.

## First Installation

```
git clone https://github.com/lieblb/testilias
cd testilias
docker-compose build
```

The last step can take up to 30 minutes on first install.

You then need to download the source code of ILIAS you want to test against and move it to `testilias/web/ILIAS`, e.g.:

```
cd /path/to/testilias
wget https://github.com/ILIAS-eLearning/ILIAS/archive/v5.3.5.tar.gz
tar xzfv v5.3.5.tar.gz
mv ILIAS-5.3.5 web/ILIAS
```

TestILIAS will instrument your ILIAS code on the first run and automatically build a fully functional installation (you will not need to perform a setup).

## Starting up TestILIAS

Starting up TestILIAS happens via the `up.py` script, which takes the number of parallel client machines you want to start:

```
cd /path/to/testilias
./up.py --n 5
```

After TestILIAS started up, you should be able to access the TestILIAS main GUI under:

`http://mymachine:11150/`

Please note that the default network setup globally exposes your port; if your firewall does not block it, other people will be able to reach your TestILIAS installation from outside (you can change this by changing TestILIAS' `docker-compose.yml`).

Be patient during the first setup, it may take some time. If your installation is local, `mymachine` will be `localhost`.

# The TestILIAS UI

## Starting Test Runs

<img src="https://github.com/lieblb/testilias/blob/master/docs/main-ui.jpg?raw=true">

Start an automatic test run via the "Start" button. "Loop" allows to to run test runs indefinitely (i.e. start a new run as soon as one ends). Workarounds gives you a list of problems currently known in ILIAS. Turning one of these checkboxes on will mean that your tests will fail sooner or later.

Clicking on the ILIAS link below the header will bring you to TestILIAS' internal ILIAS installation. You can login as root using the password "odysseus".

The results table gives you detailed protocols of each test run as well as the exported XLS. Note that all this data gets deleted permanently as soon as you hit the close button in the UI.

## Response Times

<img src="https://github.com/lieblb/testilias/blob/master/docs/response-times-ui.jpg?raw=true">

## Status

<img src="https://github.com/lieblb/testilias/blob/master/docs/status-ui.jpg?raw=true">

During test runs, TestILIAS allows you to keep track of what's happening on the various client machines:

# Technical stuff

## Debugging startup problems

Commands like `docker ps` and `docker logs testilias_master_1` are your friend.

## Recreating the docker-compose configuration

Strange things happen with docker sometimes and you want to completely recreate the complete docker-compose setup. Here's one way to do this (note that this deletes all dangling volumes of all docker containers, so be very careful if you're not a dedicated machine):

```
cd /path/to/testilias
docker-compose rm
sudo docker volume rm $(sudo docker volume ls -qf dangling=true)
docker-compose build --no-cache
```

## Cleaning up docker

Over time, and if running over several days, the involved docker images will grow larger and larger (GBs per machine). At some point,
drives might get full and ILIAS will fail with random errors. To clean up, you might want to do a `docker system prune`.

## Building the DB container from scratch

Sometimes - after purging containers - ILIAS itself won't start up and you get `An undefined Database Exception occured. SQLSTATE[42S02]: Base table or view not found`.

In these cases, delete the DB container and image, e.g.:

```
docker rm testilias_db_1
docker rmi testilias_db
```

Then start `up.py` and give it several minutes, as the DB import needs substantial time.

## Updating the DB dump for newer versions of ILIAS

TestILIAS intializes its ILIAS installation using a minimal default DB. As new ILIAS versions are published, it will be necessary to recreate this dump. Here's how to do this:

* Apply hotfixes from new ILIAS version in the TestILIAS ILIAS installation.

* Then re-export the dump:

```
docker exec -it iliasdocker_db_1 /bin/bash

> mysqldump ilias -p > dump.sql
> dev

> docker cp iliasdocker_db_1:/dump.sql /your/local/machine/testilias/db/ilias.sql
```

Now zip `ilias.sql` and update in your version control.

## Notes on the implementation

The "master" container (see `docker-compose.yml`) provides the GUI for running tests and
evaluating test runs; the frontend is just one big Javascript hack, which is not great.

The "web" and "db" container are the ILIAS web server and the ILIAS database.

Each test client runs in a dedicated Docker container, that can control one standalone
Firefox through a Selenium driver  (see "machine" in `docker-compose.yml`). The "one browser
per container" limit comes through Selenium.

An overall neater and probably faster alternative for similar projects would be running
Chrome through CEF and https://github.com/cztomczak/cefpython (which means several or all
clients could run in one Docker container).
