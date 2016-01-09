# pg [![Build Status](https://travis-ci.org/jangorecki/pg.svg?branch=master)](https://travis-ci.org/jangorecki/pg)

Postgres utilities for R language

- based on [RPostgreSQL](https://github.com/tomoakin/RPostgreSQL)
- logging powered by [logR](https://github.com/jangorecki/logR)

## Features

- wrappers on many commonly used `DBI::db*` functions, enables logging, default `conn` and other arguments
- new functions: `pgExistsSchema`, `pgTruncateTable`, `pgListTableColumns`, `pgDropSchema`, `pgListTableColumns` and potentially more in future
- data transfered over `pgWriteTable` enables transparent technical metadata stamping
- metadata for logging with logR can store custom arbitrary information
- helpers for processing batches of the data

## Installation

R
```r
library(devtools)
install_github("jangorecki/pg")
```

postgres
```r
docker run -it --rm -p 5432 -e POSTGRES_USER=r_user -e POSTGRES_PASSWORD=r_password -e POSTGRES_DB=r_db -e POSTGRES_PORT=5432 --name pg postgres:9.5
```

## Setup

```r
# Sys.setenv(POSTGRES_HOST="172.17.0.2") # use this for non-localhost postgres

suppressPackageStartupMessages({
    library(data.table)
    library(logR)
    library(pg)
})

options("pg.conn" = pgConnect(dbname = "r_db", user = "r_user", password = "r_password"))

# logR setup
meta = function(run_id=get_run_id(), r_user = as.character(Sys.info()[["user"]])[1L], r_timestamp = Sys.time(), r_fun = NA_character_, r_args = NA_character_) list(run_id=run_id, r_user=r_user, r_timestamp=r_timestamp, r_fun=r_fun, r_args=paste(r_args, collapse=","))
create_meta = list(run_id = "INTEGER", r_user = "VARCHAR(255)", r_timestamp = "TIMESTAMPTZ", r_fun = "VARCHAR(255)", r_args = "VARCHAR(255)")
options("logR.conn" = getOption("pg.conn"),
        "logR.schema" = "r_tech",
        "logR.table" = "logr",
        "logR.meta" = meta)

# restart schemas
pgDropSchema(c("r_tech","r_data"), cascade = TRUE, silent = TRUE, .log = FALSE)
options("run_id" = NULL)
options("digits.secs" = 3L)
logR_schema(meta = create_meta)
create_run_table(schema_name = "r_tech", table_name = "run", .log = FALSE)
```

## Use

```r
pgSendQuery("CREATE SCHEMA r_data;")

# loading data in batches
options("run_id" = NULL)
dt = data.table(a = 1:2, b = letters[2:1])
pgWriteTable(c("r_data","techstamp"), dt)

options("run_id" = new_run_id(.log = FALSE))
dt = data.table(a = 4:6, b = letters[6:4])
pgWriteTable(c("r_data","techstamp"), dt)

options("run_id" = new_run_id(.log = FALSE))
dt = data.table(a = 5:8, b = letters[8:5])
pgWriteTable(c("r_data","techstamp"), dt)

# data
pgReadTable(c("r_data","techstamp"))

# logs
pgReadTable(c("r_tech","logr"), .log = FALSE)
pgReadTable(c("r_tech","logr"), .log = FALSE)[, .(in_rows, out_rows), .(run_id, r_fun, r_args)]

# temporal 'latest' query using tech timestamp
pgGetQuery("SELECT DISTINCT ON (a) * FROM r_data.techstamp ORDER BY a, r_timestamp DESC;")
```

For more examples and full reproducible environment see CI workflow and [tests/tests.R](tests/tests.R). Be aware travis-ci uses 5433 port instead of default 5432, this can easily be controlled by environment variables as demonstrated in travis yaml.  

## Notes

Only postgres 9.5+ support is planned, currently there is no 9.5+ specific code yet.  
