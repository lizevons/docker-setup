# docker-setup

Generic Docker setup processes for Postgres and CloudBeaver with ports available to QGIS and R.

## Instructions for creating a starter container with an empty database

The files in this folder contains the basic building blocks for setting up a Docker container with a PostgreSQL database as well as a CloudBeaver - a web-based data management tool. In the first step of the process, we want to create a container with an empty PostgreSQL database, that we can populate with data through QGIS or R.

### Step 1: Make a copy of the following files to where you want to set up the project

**docker-compose.yaml** - contains the Docker setup instructions.

**.env** - stores database and CloudBeaver user names and passwords.

**init/00-init.sql** contains the SQL instructions for creating a database and a named schema. This file runs as part of the docker compose process.

### Step 2: Create custom user names and passwords

> [!Note]
> To avoid your browser shouting at you, it is really important to change these generic usernames and passwords!

-   Copy all the files in this folder to where you would like to set up the project

-   In **.env** set custom values for `DB_PASSWORD`, `DB_NAME`, `CLOUDBEAVER_ADMIN` and `CLOUDBEAVER_PASSWORD` - DO NOT CHANGE `DB_USER=postgres`

-   If you are creating a git repository, add .env to the .gitignore

-   In **00-init.sql** give the database a name (replace `myprojectdb`) and create a named schema (replace `myschema`). You can also delete this line if you just want to use the public schema (created by default).

### Step 3: Run docker compose

-   In the same folder where the `docker-compose.yaml` file is located, open a terminal

-   In the terminal type `docker compose -p test up` - replace `test` with the name you'd like to use for the container

-   When the process is finished, the container will be visible in docker with the two services (postgis and CloudBeaver) turned on

### Step 4: Set up a database connection in CloudBeaver

-   Click on the Port(s) link next to the CloudBeaver service in Docker, which opens CloudBeaver in your browser

-   Log in to CloudBeaver using the username (`CLOUDBEAVER_ADMIN)` and password (`CLOUDBEAVER_PASSWORD)` defined in .env

-   Create a new PostgreSQL database connection:

    -   At top right of screen, click Administration, and then +ADD

    -   Set up the database connection using the following settings

| Field | Value | Source |
|------------------------|------------------------|------------------------|
| Host | postgis | Name of postgis service defined in `docker-compose.yaml` |
| Port | 5432 | Default value |
| Database | myprojectdb | Name of database defined in `init.sql` |
| User | postgres | `DB_USER` in `.env` |
| Password | postgres | `DB_PASSWORD` in `.env` |

## Additional functions

### Connecting to database from QGIS

A QGIS connection is useful for importing data into the database, and also for visualising spatial data in the database. One thing to note is that QGIS expects the postgis extension to be in the public schema. If it is in a different schema, or not set up, in Cloudbeaver, run the following SQL:

| Situation | Connection | SQL |
|------------------------|------------------------|------------------------|
| No postgis extension | public\@myprojectdb | CREATE EXTENSION postgis; |
| Postgis not in public | public\@myprojectdb | ALTER USER postgres SET search_path TO myschema,public,"\$user"; |

#### Steps

1.  In QGIS select Layer \> Data source manager

2.  Select PostgreSQL and click New

3.  In the connection form, add the following

| Field    | Value                                      |
|----------|--------------------------------------------|
| Name     | Give it any name                           |
| Service  | Leave blank                                |
| Host     | localhost                                  |
| Port     | 6005 (as defined in `docker-compose.yaml`) |
| Database | myprojectdb (as defined in `init.sql`)     |

Leave Authentication on No Authentication.

Select 'Also list tables with no geometry' and 'Use estimated table metadata'

When asked for username and password, use the `DB_USER` and `DB_PASSWORD` as set up in `.env`

### Connecting to database from R

Database connections from R enables analyses of data sourced directly from the PostgreSQL database (no need to export data).

In the root folder of the R project, create an `.Renviron` file (add it to .gitignore if project is part of a git repository). Add to `.Renviron` the following variables:

```         
DB_NAME=myprojectdb (as defined in init.sql)
DB_HOST=localhost
DB_PORT=6005 (as defined in docker-compose.yaml)
DB_USER=postgres (DB_USER in .env)
DB_PASSWORD=postgres (DB_PASSWORD in .env)
```

In the R script where you want to connect to the database, source the script `db-connection.R` (included in this repository), using the following code:

``` r
## Connect to the database OUTSIDE the container (local testing).
source("db-connection.R")
```

### Backing up the database

1.  In Docker Desktop, click on the name of the database image in the container (postgis).

2.  From among the tabs at the top of the screen, select **Exec**, this will open Docker’s internal Terminal for that container.

3.  It should say `root@...:/#`, if not, type **bash** and press enter.

4.  In the Docker terminal, enter the following pg command:

| Command | What it does |
|------------------------------------|------------------------------------|
| pg_dump | create a SQL dump (backup file) |
| -h localhost | connect to PostgreSQL via localhost |
| -p 5432 | use port **5432**, the default PostgreSQL port. |
| -U postgres | Connect as the **user `postgres`**. |
| -d myprojectdb | Connect to the **database named `myprojectdb`**. |
| -n myschema | OPTIONAL - dump only a specific schema |
| --create | Forces a CREATE DATABASE statement to be included |
| -F p | Forces dump to be written in plain SQL format |
| -v | Verbose mode – print progress and debugging information during the dump. |
| --no-owner | Do **not include ownership information** in the dump (i.e., no `ALTER OWNER` commands). Useful when restoring to a different user. |
| --clean | Drop and recreate objects during restore - avoids clashes with schemas or other objects that may already exist |
| -f /var/lib/postgresql/data/db-backup.sql | Docker's internal path to the container volume, plus the filename for the SQL dump file |

You can copy the full command here:

``` bash
pg_dump -h localhost -p 5432 -U postgres -d myprojectdb --create -F p -v --no-owner --clean -f /var/lib/postgresql/data/db-backup.sql
```

#### Retrieve the backup from the Docker volume

In Docker Desktop, click on Volumes, and then find the container's volume. Its name will be the name of the container (in this case test), underscore, name of postgis image (in this case postgis), followed by data. For example: `test_postgis_data`. Find the backup file, right click on it, and select "Save as...", browse to where you want to store the backup. If you leave the backups in the volume, they risk being deleted when the container is deleted.

### Deleting the docker container

1.  In the same folder where the `docker-compose.yaml` file is located, open a terminal

2.  In the terminal type:

``` bash
docker-compose -p test down --volumes
```

Replace `test` with the name of the container. The `--volumes` command also deletes the data associated with the container, so make sure that it is backed up before removing.

## Instructions for creating a container that restores an existing database

Once you are done building the database, you can make your Docker container system portable - e.g. allow other users to set up a copy of your database, using the following process:

1.  Make a database backup using the instructions above.

2.  Save the backup file in the init folder - together with 00-init.sql.

3.  Run `docker-compose up`

> [!Warning]
> Because the pg_dump was created with a `--clean` statement, this process can potentially overwrite an existing database if you create a new container with the same name as an existing container. Implement with care!
