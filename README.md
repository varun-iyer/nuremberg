# HLS Nuremberg Trials Project

> This is a Django client for the digital archives of the Nuremberg Trials
> Project maintained by the Harvard Law Library.  It is intended as a
> open-access web app for scholars, researchers, and members of the public,
> exposing the digitized documents, full-text trial transcripts, and rich
> search features for the same in a friendly, modern interface.

## Running the client locally:

The client uses docker.

1. `docker-compose up`
2. `mv dumps/nuremberg_prod_dump_2022-08-02.sqlite3.zip . && unzip nuremberg_prod_dump_2022-08-02.sqlite3`
3. `mv nuremberg_prod_dump_2022-08-02.sqlite3 nuremberg_dev.db`
4. `docker-compose cp solr_conf/ solr:/opt/solr-8.11.2/solr_conf`
5. `docker-compose exec cp /opt/solr-8.11.2/solr_conf /var/solr/data/nuremberg_dev`
6. `docker-compose exec solr solr create_core -c nuremberg_dev -d solr_conf`
7. `docker-compose exec web python manage.py rebuild_index`
7. `docker-compose exec web python manage.py runserver`


## Project Structure

The project is organized into several feature-oriented modules ("apps" in
Django parlance). Each module includes all URL routing, model and view code,
tests, templates, JavaScript code, and static assets for the corresponding
feature set:

- `nuremberg`: Top-level namespace for organizational purposes only.  `.core`:
- Top-level URL routing, test frameworks, base templates and middleware, and
- site-wide style files.  `.settings`: Environment-specific Django settings
- files.  `.content`: Files for static pages with project information, etc.
- `.documents`: Files for displaying digitized document images.
- `.transcripts`: Files for full-text transcripts and OCR documents.
- `.photographs`: Files for displaying images from the photographic archive.
- `.search`: Files for the search interface and API.

    This document covers the following topics:

- [Setting up a development environment](#setup) [Running the test
- suite](#testing) [Configuring project settings](#project-settings)
- [Data](#data) [Packaging static assets](#static-assets)

#### Solr

For the latest Solr 8, build the Solr schema by running:

```
bash docker-compose exec web python manage.py build_solr_schema
--configure-dir=solr_conf
```

This will generate both `schema.xml` and `solrconfig.xml` under the `solr_conf`
directory. You can then rebuild the Solr docker container to use the updated
config files.

To update the index itself:

```
bash manage.py rebuild_index
```

(It will take a couple of minutes to reindex fully.)

Do this any time you make changes to `search_indexes.py` or `schema.xml`.

#### Run the server

You should now be all set to run the local server:

```
bash python manage.py runserver
```

Then visit [localhost:8000](http://localhost:8000).

### Testing

Tests in the project are generally high-level integration acceptance tests that
exercise the full app stack in a deployed configuration. Since the app has the
luxury of running off of a largely static dataset, the test database is
persistent, greatly speeding up setup and teardown time.

#### Running tests

Make sure you have installed test dependencies and initialized the test
database in [Setup](#setup) above. Then simply:

```
bash py.test
```

Pytest is configured in `pytest.ini` to run all files named `tests.py`.

There is also a Selenium suite to test the behavior of the document viewer
front-end. These tests take a while to run, don't produce coverage data, and
are relatively unreliable, so they aren't run as part of the default suite.
However they are still useful, as they exercise the full stack all the way
through image downloading and preloading. They can be run explicitly when
necessary.

The browser tests require use of the Docker development environment, which
includes an isolated Selenium container for running the tests.

```
bash docker-compose exec web pytest nuremberg/documents/browser_tests.py
```


#### Coverage

Running the test suite will print a code coverage report to the terminal, as
well as an interactive HTML report in `coverage/index.html`. Template code is
included in the report. Coverage settings are configured in `.coveragerc`.

> NOTE: There is an open issue
> [#25](https://github.com/nedbat/django_coverage_plugin/issues/25) with
> django_coverage_plugin which will hide certain warnings related to template
> coverage settings, thus the requirement of the
> [emmalemma@e083da1](https://github.com/emmalemma/django_coverage_plugin/commit/e083da1)
> fork. The plugin works fine, but if you see 0% coverage in templates,
> double-check your debug settings.

#### Pre-commit hook

To improve test compliance, there is a git pre-commit hook to run the test
suite before each commit. It's self-installing, so just run:

```bash bash ./nuremberg/core/tests/pre-commit-hook.sh ```

Now if any test fails, or test coverage is below 95%, the hook will cancel the
commit.

### Project Settings

> NOTE: An example configuration used for the demo site on Heroku is in the
> [heroku](https://github.com/harvard-lil/nuremberg/tree/heroku) branch as
> `staging.py`.

Environment-specific Django settings live in the `nuremberg/settings`
directory, and inherit from `nuremberg.settings.generic`. The settings module
is configured by the `DJANGO_SETTINGS_MODULE` environment variable; the default
value is `nuremberg.settings.dev`.

Secrets (usernames, passwords, security tokens, nonces, etc.) should not be
placed in a settings file or committed into git. The proper place for these is
an environment variable configured on the host, and read via `os.environ`. If
they must live in a `.py` file, they should be included in the environment
settings file via an `import` statement and uploaded separately as part of the
deployment process.

(The only exception to this is the defaults used in the dev environment.)


### Data

Since it is expected that this app will host a largely static dataset, there
isn't an admin interface to speak of. Updates can go straight into MySQL. Just
ensure that any changes are reindexed by Solr.

The test fixture database dump is, for all intents and purposes, a
production-ready database at the time of this writing. However, that file
should only be updated when necessary to support testing new features.

#### Solr

Solr indexes are defined in relevant `search_indexes.py` files, and additional
indexing configuration is in the
`search/templates/search_configuration/schema.xml` file used to generate
`solr_conf/schema.xml`.

The Solr schema must be maintained as part of the deploy process. When
deploying an updated schema, make sure to generate and upload the `schema.xml`
and `solrconfig.xml` files created by using `manage.py build_solr_schema
--configure-dir=solr_conf`, then run a complete reindexing.

> WARNING: Be cautious when doing this in production-- although in general
> reindexing will happen transparently, certain schema changes will cause a
> `SCHEMA-INDEX-MISMATCH` error that will cause search pages to crash until
> reindexing completes.

#### Reindexing

If writes are relatively infrequent, manual reindexing using `manage.py
update_index` should work fine. If writes happen relatively often, you should
set up a `cron` script or similar to automate reindexing on a nightly or hourly
basis using `--age 24` or `--age 1`. (Note: This will restrict reindexing only
for indexes that have an `updated_at` field defined; currently, `photographs`
does not, but completely reindexing that model is fast anyway.)

Even a full reindex should only take a few minutes to run, and the site can
continue to serve requests while it happens. For more fine-grained information
on indexing progress, use `--batch-size 100 --verbosity 2` or similar.

#### Transcripts

There is a management command `manage.py ingest_transcript_xml` which reads a
file like `NRMB-NMT01-23_00512_0.xml` (or a directory of such files using `-d`)
and generates or updates the appropriate transcript, volume, and page models.
Since some values read out of the XML are stored in the database, re-ingesting
is the preferred way to update transcript data. If database XML is modified
directly, call `populate_from_xml` on the appropriate TranscriptPage model to
update date, page, and sequence number.

Remember to run `manage.py update_index transcripts` after ingesting XML to
enable searching of the new content.

### Static Assets

#### CSS

CSS code is generated from `.less` files that live in
`nuremberg/core/static/styles`. The styles are built based on Bootstrap 3
mixins, but don't bundle any Bootstrap code directly to ensure a clean semantic
design.

Compilation is handled automatically by the `django-static-precompiler` module
while the development server is running.

#### JavaScript

JavaScript code is organized simply. JS dependencies are in
`core/static/scripts`, and are simply included in `base.html`. App code is
modularized using `modulejs`, to ensure that only code in the module defined in
the relevant template is run.

The only significant JavaScript app is the document image loading, panning, and
zooming functionality implemented in `documents/scripts`. That functionality is
organized as a set of Backbone.js views and viewmodels. There is a smaller
amount of code in `transcripts` to handle infinite scrolling and page
navigation, which is implemented in pure jQuery. There are also a handful of
minor cosmetic features implemented in `search`.

In production, all site javascript is compacted into a single minified blob by
`compressor`. (The exception is the rarely-needed dependency `jsPDF`.)

#### In Production

When deploying, you should run `manage.py compress` to bundle, minify and
compress CSS and JS files, and `manage.py collectstatic` to move the remaining
assets into `static/`. This folder should not be committed to git.

For deployment to Heroku, these static files will be served by the WhiteNoise
server. In other environments it may be appropriate to serve them directly with
Nginx or Apache. If necessary, the output directory can be controlled with an
environment-specific override of the `STATIC_ROOT` settings variable.


## Docker Development Environment

We have initial support for local development via `docker-compose`.

### Initial Setup

docker-compose up -d --build

This will build a 1.07GB image for the Nuremberg app and a 573MB image for the
solr index, which may take a few minutes.

./init.sh

This will set up the development and test databases and populate the solr
index. The database initializes quickly; the solr index takes several minutes.

### Subsequently

Start up the Docker containers in the background:

docker-compose up -d

Fire up the web server:

docker-compose exec web ./manage.py runserver 0.0.0.0:8000

Then visit [localhost:8000](http://localhost:8000). Press `Ctrl-c` to quit the
web server.

Run the python tests:

`docker-compose exec web pytest`

Run the browser tests:

`docker-compose exec web pytest nuremberg/documents/browser_tests.py`

(These tests run in a separate Selenium docker container, so no further setup
should be required.)

### Starting Fresh

Take down the Docker containers:

$ docker-compose down

Remove the databases and the solr index:

docker volume rm nuremberg_db_data nuremberg_solr_collection

Remove the images:

docker rmi nuremberg nuremberg-solr
