## Bigmetadata

### What it is

Bigmetadata comprises four parts:

    ETL | Observatory | Metadata | Catalog

Everything is dockerized and can be run on a standalone EC2 instance provided
Docker is available.

* __ETL__: Luigi tasks that extract data from anywhere on the web, transform the
  data in modular and observable steps, generate metadata, and upload
  observatory datasets in the process.  While the ETL pushes transformed
  datasets to CartoDB, it actually is backed by its own separate
  Postgres/PostGIS database.

* __Observatory__: transformed and standardized datasets living on a [CartoDB
  instance](https://observatory.cartodb.com).  This is where we can pull our
  actual data from, whether shuffling bytes or using a FDW.  This is also where
  preview visuals using widgets or other CartoDB JS interfaces can get their
  underlying data from.

* __Metadata__: human and machine-readable descriptions of the data in the
  observatory.  Currently structured as
  [a large hierarchy of JSON blobs](https://github.com/talos/bmd-data), but
  I am moving towards a simpler tabular relational store in postgres based off
  of `information_schema`.  While tables have some minimal metadata, for the
  most part what is described are columns in tables.

* __Catalog__: a [static HTML guide](https://cartodb.github.io/bigmetadata) to
  data in the observatory generated from the metadata.  Docs are generated
  using [Sphinx](http://sphinx-doc.org/) and hosted on GitHub pages.


### Installation

You'll need [docker](https://www.docker.com/) and [docker-compose](https://docs.docker.com/compose/) to get
started.

You'll then need to configure `CARTODB_API_KEY` and `CARTODB_URL` in the
`.env` file in order to upload to CartoDB.

    cp .env.sample .env
    vim .env

Most of the requirements are available as images, but you will need
to build the postgres and bigmetadata containers before gettings tarted:

    docker-compose build

Then get all your containers running in the background:

    docker-compose up -d

### Tasks

Most of the common tasks have already been wrapped up in the `Makefile`.

* `make catalog`: Regenerate the catalog
* `make pdf-catalog`: Regenerate the PDF version of the catalog
* `make deploy-catalog`: Deploy the catalog to Github Pages. This will also
  rebuild both the HTML and PDF catalogs
* `make acs`: ETL the entirety of the American Community Survey (Census)
* `make tiger`: ETL the boundary files of the census
* `make sh`: Drop into the bigmetadata container to run shell scripts
* `make python`: Drop into an interactive Python shell in the bigmetadata
  container
* `make psql`: Drop into an interactive psql session in the bigmetadata
  container's database
* `make sync`: Sync local data and metadata to the Observatory account.

Any other task can be run using `docker-compose`:

    docker-compose run bigmetadata luigi --module tasks.path.to.task \
      TaskName --param1 val1 --param2 val2

For example, to run QCEW numbers for one quarter:

    docker-compose run bigmetadata luigi --module tasks.us.bls QCEW --year 2014 --qtr 4

### Naming conventions

#### Tables

In bigmetadata's postgres, tables live in the `observatory` schema, with an
auto-generated hash for the tablename prefixed by `obs` (EG `obs_<hash>`).
In `obs_table`, the table ID refers back to the task which created that table
as well as the parameters called.

#### Columns

Column IDs in metadata are fully-qualified, like tables, such as
`us.bls.avg_wkly_wage_trade_transportation_and_utilities`.  In tables
themselves, column names are the `colname` from metadata,
so the above would be `avg_wkly_wage_trade_transportation_and_utilities` in
an actual table.

### Metadata

#### Why favor descriptions of columns over tables in metadata?

Our users need third party data in their maps to make it possible for them to
better interpret their own data.

In order to make this as simple as possible, we need to rethink the prevailing
model of finding external data by where it's sourced from.  Instead, we should
think of finding external data by need.

For example, instead of requiring our user to think "I need race or demographic
data alongside my data, I'm in the US, so I should look at the census", we
should enable our user to look up race or demographic data -- and figure out
which columns they need from the census without particularly worrying about
delving into the source.
