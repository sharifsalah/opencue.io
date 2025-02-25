---
title: "Setting up the database"
linkTitle: "Setting up the database"
weight: 1
date: 2019-02-22
description: >
  Set up the OpenCue database
---

This page describes how to install and set up a
[PostgreSQL](https://www.postgresql.org/) database for use with OpenCue.

OpenCue supports a single database per deployment, which serves as the single source of truth
within your deployment.

## System requirements

[Cuebot](/docs/getting-started/deploying-cuebot) servers are the only
component that access the database directly. All other components, such as
[PyCue](/docs/getting-started/installing-pycue-and-pyoutline), interact with the
database indirectly via the Cuebot's gRPC API. For this reason, make sure Cuebot
has a low-latency connection to the database, either by running both on the same
machine or on the same local network.

## Before you begin

To follow the instructions in this guide, you'll need the following software:

*   [Python](https://www.python.org/)
*   [pip](https://pypi.org/project/pip/) Python package manager

If you're on macOS, you'll also need [Homebrew](https://brew.sh/).

## Installing Postgres

Before you set up the database, make sure you complete the steps to install
PostgreSQL.

### Installing on Docker

1.  First, pull the Postgres image from Docker hub:

    ```shell
    docker pull postgres
    ```

1.  Then, start your Postgres container:

    ```shell
    export PG_CONTAINER=postgres
    docker run -td --name $PG_CONTAINER postgres
    ```

1.  Create a superuser named after your current OS user, which is used for the
    rest of the admin commands in this guide.

    ```shell
    docker exec -it --user=postgres $PG_CONTAINER createuser -s $USER
    ```

1.  To make things easier, install a Postgres client:

    {{% alert color="warning" title="Important" %}}Run the following commands
    *on the host machine*.{{% /alert %}}

    On yum-based Linux you can run

    ```shell
    yum install postgresql-contrib
    ```

    On macOS, the easiest way is with Homebrew:

    ```shell
    brew install postgresql
    ```

1.  Finally, export the `DB_HOST` environment variable, which is used in the
    rest of this guide by fetching the IP address of your Postgres container:

    {{% alert color="warning" title="Important"%}}Run the following commands
    *on the host machine*.{{% /alert %}}

    ```shell
    export DB_HOST=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $PG_CONTAINER)
    ```

### Installing on Linux

The following steps assume a `yum`-based Linux distribution, such as
[CentOS](https://www.centos.org/).

See [the Postgres docs](https://www.postgresql.org/download/) for instructions
for other distributions.

1.  First, run `yum` to install the required Postgres packages:

    ```shell
    yum install postgresql-server postgresql-contrib
    ```

1.  Next, initialize your Postgres installation and configure it to run as a
    service:

    ```shell
    postgresql-setup initdb
    systemctl enable postgresql.service
    systemctl start postgresql.service
    ```

1.  Create a superuser named after your current OS user, which is used for the
    rest of the admin commands in this guide:

    ```shell
    su -c "createuser -s $USER" postgres
    ```

1.  Finally, export the `DB_HOST` environment variable, which is used in the
    rest of this guide:

    ```shell
    export DB_HOST=localhost
    ```

### Installing on macOS

For macOS, we recommend installing PostgreSQL using
[Homebrew](https://brew.sh/).

To install the database:

1.  Install PostgreSQL:

    ```shell
    brew install postgresql
    ```

1.  Then, start the PostgreSQL software:

    ```shell
    brew services start postgres
    ```

1.  Finally, export the `DB_HOST` environment variable, which is used in the
    rest of this guide:

    ```shell
    export DB_HOST=localhost
    ```

## Creating the database

After you've installed PostgreSQL, you must create a database.

This assumes your current OS user ($USER) is a db superuser of the same name,
and that you are able to log in without a password.

To create a database:

1.  Define and export the following shell variables for a database named
    `cuebot_local`:

    ```shell
    export DB_NAME=cuebot_local
    export DB_USER=cuebot
    export DB_PASS=<changeme>
    ```

1.  Create the database and the user that Cuebot uses to connect to it:

    ```shell
    createdb $DB_NAME
    createuser $DB_USER --pwprompt
    # Enter the password stored in DB_PASS when prompted
    psql -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO $DB_USER" $DB_NAME
    ```

    **If you are running Postgres in a Docker container**, you will instead need
    to run the same commands from within that container:

    ```shell
    docker exec -it --user=postgres $PG_CONTAINER createdb $DB_NAME
    docker exec -it --user=postgres $PG_CONTAINER createuser $DB_USER --pwprompt
    # Enter the password stored in DB_PASS when prompted
    docker exec -it --user=postgres $PG_CONTAINER psql -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO $DB_USER" $DB_NAME
    ```

    {{% alert color="warning" title="Important" %}}`ALTER DEFAULT PRIVILEGES` will
    grant privileges for any tables created **in the future**. If you need to
    revisit this step after your database has already been populated, you can run
    the following similar command to grant privileges on all **existing**
    tables:

    ```shell
    psql -c "GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO $DB_USER" $DB_NAME
    ```
    {{% /alert %}}

## Populate the database

### Option 1: Download the published schema

Visit https://github.com/AcademySoftwareFoundation/OpenCue/releases and download the SQL files
from the latest release's Assets. There should be two - a `schema` file and a
`demo_data` file.

To populate the database:

1.  To populate the database schema and some initial data, run the `psql`
    command:

    ```shell
    psql -h $DB_HOST -f <path to schema SQL file> $DB_NAME
    ```

    To see a list of flags for the `psql` tool, run the `psql --help` command.
    For example, if your database is running on a remote host, specify the `-h`
    flag. If you need to specify a different username, specify the `-U` flag.

1.  The `demo_data` SQL file contains a series of SQL commands to insert some
    sample data into the Cuebot database, which helps demonstrate various
    features of the software. To execute the SQL statements, run the `psql`
    command:

    ```shell
    psql -h $DB_HOST -f <path to demo_data SQL file> $DB_NAME
    ```

### Option 2: Apply migrations from source

The OpenCue source code stores its database schema as *migrations*, which are a
sequence of SQL files which the database administrator can apply incrementally
to construct the full database schema.

While not necessary for a demo environment, we recommend this method for most
production deployments of OpenCue use. Any future changes to the PostgreSQL
schema will be published as new migrations, which allows your database
administrator to safely apply the schema changes without needing to do a full
rebuild of the database.

You need to
[check out the source code](/docs/getting-started/checking-out-the-source-code). The rest
of this section assumes your current directory is the root of the checked out
source.

Migrations are published as raw SQL, so it should be possible to use your
database management tool of choice for applying the migrations. This example
uses [Flyway](https://flywaydb.org/).

To apply the migrations:

1.  On macOS, the easiest way to install Flyway is via Homebrew:

    ```shell
    brew install flyway
    ```

    For installation instructions for other platforms, see the
    [the Flyway documentation](https://flywaydb.org/documentation/commandline/#download-and-installation).

1.  Run the `flyway` command to execute the migrations:

    ```shell
    flyway -url=jdbc:postgresql://$DB_HOST/$DB_NAME -user=$USER -n -locations=filesystem:cuebot/src/main/resources/conf/ddl/postgres/migrations migrate
    ```

1.  The OpenCue repository also includes the PostgreSQL `demo_data.sql` file.
    This file contains a series of SQL commands to insert some sample data into
    the Cuebot database, which helps demonstrate various features of the
    software. To execute the SQL statements, run the `psql` command:

    ```shell
    psql -h $DB_HOST -f cuebot/src/main/resources/conf/ddl/postgres/demo_data.sql $DB_NAME
    ```

    To see a list of flags for the `psql` tool, run the `psql --help` command.
    For example, if your database is running on a remote host, specify the `-h`
    flag. If you need to specify a different username, specify the `-U` flag.

## What's next?

*   [Deploying Cuebot](/docs/getting-started/deploying-cuebot)
