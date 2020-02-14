# TestArchiver
TestArchiver is a tool for archiving your test results to a SQL database.

And [Epimetheus](https://github.com/salabs/Epimetheus) is the tool for browsing the results you archived.

# Testing framework support

| Framework | Status | Fixture test status | Parser option |
| --- | --- | --- | --- |
| Robot Framework | [Supported](robot_tests/) | Done | robot
| Mocha | [Supported](mocha_tests/) | Done | mocha-junit |
| JUnit | Experimental | Missing | junit |
| xUnit | Experimental | Missing | xunit |
| MSTest | Experimental | Missing | mstest |
| pytest | [Supported](pytest/) | Done| pytest-junit |

Experimental status here means that there is a parser that can take in e.g. generic JUnit formatted output but there is no specific test set or any extensive testing or active development for the parser.

Contributions for output parsers or listeners for different testing frameworks are appreciated. Contributing a fixture test set (that can be used to generate output files for developing a specific parser) is helpful for any new framework.

# Supported databases

## SQLite
[SQLite](https://www.sqlite.org) default database for the archiver and is mainly useful for testing and demo purposes. Sqlite3 driver is part of the python standard library so there are no additional dependencies for trying out the archiver.

## PostgreSQL
[PostgreSQL](https://www.postgresql.org) is the currently supported database for real projects. For example [Epimetheus](https://github.com/salabs/Epimetheus) service uses a PosrgreSQL database. For accessing PostgreSQL databases the script uses psycopg2 module: `pip install psycopg2` or `pip install psycopg2-binary`

# Basic usage

The output files from different testing frameworks can be parsed into a database using `test_archiver/output_parser.py` script.

```
python3 test_archiver/output_parser.py --database test_archive.db output.xml
```
Assuming that `output.xml` is a output file generated by Robot Framework (the default parser option), this will create a SQLite database file named `test_archive.db` that contains the results.

For list of other options: `python3 test_archiver/output_parser.py --help`
```
  --config CONFIG_FILE  path to JSON config file containing database
                        credentials
  --dbengine DBENGINE   Database engine, postgresql or sqlite (default)
  --database DATABASE   database name
  --host HOST           databse host name
  --user USER           database user
  --pw PW, --password PW
                        database password
  --port PORT           database port (default: 5432)
  --dont-require-ssl    Disable the default behavior to require ssl from the
                        target database.
  --format {robot,robotframework,xunit,junit,mocha-junit,pytest-junit,mstest}
                        output format (default: robotframework)
  --repository REPOSITORY
                        The repository of the test cases. Used to
                        differentiate between test with same name in different
                        projects.
  --team TEAM           Team name for the test series
  --series SERIES       Name of the testseries (and optionally build number
                        'SERIES_NAME#BUILD_NUM')
  --metadata METADATA   Adds given metadata to the testrun. expected_format
                        'NAME:VALUE'
  --change_engine_url CHANGE_ENGINE_URL
                        Starts a listener that feeds results to ChangeEngine
```

## Data model
[Schema and data model](test_archiver/schemas/)


## Useful metadata
There are meta data that are useful to add with the results. Some testing frameworks allow adding metadata to your test results and for those frameworks (e.g. Robot Framework) it is recommended to add that metadata already to the tests so the same information is also available in the results. Additional metadata can be added when parsing the results using the `--metadata` option. Metadata given during the parsing is linked to the top level test suite.

`--metadata NAME:VALUE`

## Test series and teams
In the data model, each test result file is represented as single test run. These test runs are linked and organized into builds in in different result series. Depending on the situation the series can be e.g. CI build jobs or different branches. By default if no series is specified the results are linked to a default series with autoincrementing build numbers. Different test runs (from different testing frameworks or parallel executions) that belong together can be organized into the same build. Different test series are additionally organized by team. Series name and build number/id are separated by `#`.

Some examples using the `--series` and `--team` options of `output_parser.py`
* `--series ${JENKINS_JOB_NAME}#${BUILD_NUMBER}`
* `--series "UI tests"#<commit hash>`
* `--series ${CURRENT_BRANCH}#${BUILD_ID} --team Team-A`
* `--series manually_run`


Each build will have a build number in the series. If the build number is specified then that number is used. If the build number/id is omitted then the build number will be checked from the previous build in that series and incremented. If the build number/id is not a number it is considered a build identifier string. If that id is new to the series the build number is incremented just as if it no build number was specified. If the same build id is found in the same test series then the results are added under that same previously archived build.

If the tests are executed in a CI environment the build number/id is an excellent way to link the archived results to the actual builds.

The series can also be indicated using metadata. Any metadata with name prefixed with `series` are interpreted as series information. This is especially useful when using listeners. For example when using Robot Framework metadata `--metadata team:A-Team --metadata series:JENKINS_JOB_NAME#BUILD_NUMBER`
