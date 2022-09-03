# Great Expectations

## **Data integrity checks with Great Expectations**


[Great Expectations](https://greatexpectations.io) (GE) is a powerful python library to perform data integrity and data quality checks.

We are performing several integrity checks using GE. Two main integrity checks are performed, a check for totals and an integrity check over columns. After the checks run, two pandas DataFrames are generated and appended to their assigned tables in the SQL DB (`great_expectations_item_expectations`  for expectations and `great_expectations_totals_dataset` for totals.).


All checks are performed sequentially by browsing to the `/vendor/great_expectations` path in the Gobierto Contratos repo and running the `run_checks.py` script.


The `run_checks.py` script goes through the following steps:



1. Run the context script
   * Loads the `config.yml` file
   * Connects to the database running a script that generates the connection string and passing it to `sqlalchemy.create_engine`
   * Saves the date for `yesterday`, `today`, `tomorrow` and the time when the script is ran `now`
   * Imports expectations configuration from the `expectations` directory containing the expectations for each table in yaml config files. The tables for which the expectations are loaded are defined by the `expectations_tables` key in the `config.yml` file.
2. Run the functions script
   * Defines all the functions that are used throughout the script, each function contains a docstring explaining their purpose, parameters and what the function returns.
3. The queries are performed (using `run_queries`) and saved to a `dfs` variable (note this is a dictionary of dataframes). These tables are also those defined in the `expectations_tables` key in the `config.yml` file.
4. The unit tests for the queries are performed using the dataframes just obtained in the previous step.
5. Define the parameters for the next functions in a dictionary
6. Generate the totals table using the `generate_totals_table` function
7. Run the expectations using the `run_checks` function
8. Generate the table with the items that failed the checks using the `generate_checks_results_table` function

   \

## **YAML files**

Most of the configuration is setup in yaml files, there's the `config.yml`, the yamls with the expectations' configuration and parameters (under the `expectations` directory) and the `query_unit_tests.yml` file under the `expectations` directory which defines the queries for the unit tests performed on the initial data which is checked using the expectations (the dataframes obtained from the `run_queries` queries function in step 3).

### **config.yml**

The most important configurations (aside from the expectations) are stored in this file. Here's a description of each parameter:



1. `tables_to_check` are lists of table names (as contained in the database) where checks are performed, as well as methodology for running checks over the tables.
   * `expectations_tables` refers to tables the expectations configured in the expectations dir for which the checks will be performed
   * ⭐ `totals_tables` refers to tables for which their total counts will be logged
   * `run_table_by_table`  determines how the checks should be performed, whether running the process for each able individually or to run the process for all tables together.
     * if `run_table_by_table`  is True, the checks are run individually table by table (more memory efficient but minimally slower, as items are cleared from memory every time a table is done being checked).
     * if `run_table_by_table`  is False, the checks are performed on all tables together (still individually per table, but perhaps minimally faster as functions dont need to be re-ran individually per table)

       \
2. `config_params` are general configuration parameters for the process
   * ⭐ `date_from_where_to_check` refers to the date from where to run the checks, meaning all queries will be applied the filter 'updated_at > date_from_where_to_check' in their WHERE clause
     * if `date_from_where_to_check` is 'all', it'll check since the beggining of time
     * if `date_from_where_to_check` is 'yesterday', it'll check since the same time as the checks start but the previous day
     * if `date_from_where_to_check` is 'today', it'll check since 'now'
     * otherwise, any date that can be used to compare against a date field in the database can be inputted here. i.e. '2021-05-01'

   \
   * `modify_db_tables` refers to whether the checks/totals should (or not) be pushed to the SQL db
     * if `modify_db_tables` is True, totals and expectations will be pushed to the database
     * if `modify_db_tables` is False, totals and expectations will NOT be pushed to the database

       \
   *  ⭐ `db_insert_chunksize` refers to the chunksize for pandas' `to_sql` method. This parameter is **also used** to determine the chunksize when performing deletes in the `compare_checked_items_with_current` function.

     \
   * `db_table_for_expectations` refers to the table to push the checks' results to in the SQL db

     \
   * `db_table_for_expectations_column_names` refers to the column names for the DB table db_table_for_expectations, must match with database names (has to be a list of 5 elements like: `['date', 'item_id', 'table', 'check', 'passed']`)

     \
   * `db_table_for_totals` refers to the table to push the totals table, for which the total counts per table in the previous totals_tables params list specified under tables_to_check

     \
   * `db_table_for_totals_column_names` refers to the column names for the DB table db_table_for_totals, must match with database names (has to be a list of 3 elements like: `['date', 'table', 'count']`)

     \
   * `db_table_for_expectations_counts` refers to the table to push each GE checks' item counts to in the SQL db.

     \
   * `db_table_for_expectation_counts_column_names` refers to the column names for the DB table db_table_for_expectation_counts, it must match with database column names.

     \
   *  ⭐ `remove_successful_items_from_db` Refers to whether to remove or keep elements currently in the database that passed any checks (but had previously failed at least one check)
     * if `remove_successful_items_from_db` is True, then the items that passed but had previously failed are removed from the SQL DB table
     * if `remove_successful_items_from_db` is False, then the items that passed but had previously failed are NOT removed from the SQL DB table

       \

   `general_expectations_params` are global parameters that will be passed to all expectations when executed
   * `result_format` great expectations result format, described [here](https://docs.greatexpectations.io/docs/reference/expectations/result_format/)
     * `'COMPLETE'` returns all the row ids for which the checks failed



3. `danger_zone` are parameters that should only be edited if you absolutely know what you’re doing. These may allow performing sensitive database table operations.
   * `truncate_when_everything_is_checked` will allow the GE checks script to truncate the table specified in the parameter `db_table_for_expectations` whenever the parameter `date_from_where_to_check = all`. This parameter should only be used when a specification in any expectation is changed, if this parameter is True, the process will remain halted until there's confirmation that the process should continue, as checks of all data are often only performed manually.


Parameters marked with ⭐ can be overridden when running the script in console using an appropriate flag. See “**Console parameter override when running** `run_checks.py` **script**” section below for instructions on overriding the parameters.


### **Expectations yamls structure**

The yaml files in the `expectations` directory have a specific structure, each file corresponds to a table in the database, the name of the config file **must match** the ones with the SQL DB table name for which the check is performed.


These files are used to construct the `expectations` dictionary, a dictionary where the keys are the table names and the values are dictionaries containing each corresponding yaml file's key:value pairs.

#### **Parameters for each check**

The parameters a check may have are:

* *preproc*: represents a series of python commands which are a pre-processing step (i.e. create other columns to compare with) to the check
* *filtered_df*: step where the DataFrame is filtered prior to the check. For example, for the tenders check `final_amount_0.4_initial_amount` the DataFrame needs to be filtered to only contain tenders for lots.
* *expectations*: great expectations expectation method to apply to the DataFrame corresponding to the table being checked (or the one defined on the `filtered_df` step)
* *params*: parameters to pass to the great expectations method used in the *expectation* field

  \

**For example**, the check `final_amount_0.4_initial_amount` has the following parameters:

```yml
  final_amount_0.4_initial_amount:
    preproc:
      steps:
        0: tenders['initial_amount_compare'] = tenders['initial_amount']*0.4
    filtered_df:
      name: tenders_lots
      steps:
        0: tenders_lots = tenders[tenders['number_of_batches'] > 1]
    expectation: expect_column_pair_values_A_to_be_greater_than_B
    params:
      column_A: final_amount
      column_B: initial_amount_compare
      ignore_row_if: either_value_is_missing
      or_equal: true
```


The steps the `run_checks` will follow using the information provided in this yaml entry are the following:



1. A *preproc* step is ran, where the column `initial_amount_compare` is added to the `tenders` DataFrame. This column is the product of all the items in the `initial_amount` column of the `tenders` DataFrame times `0.4`.
2. A *filtered_df* step is ran, where the DataFrame `tenders_lots` is created, only containing elements in the `tenders` table whose value for `number_of_batches` is larger than 1.
3. The `expect_column_pair_values_A_to_be_greater_than_B` *expectation* is ran passing the parameters described in the *params* key to it using unpacking (passing a python dictionary's key:value pairs as parameters=values to the function).


## **Unit tests over the queries performed**

To ensure that the queries under the `queries` directory pull the right amount of contracts, tenders, etc. from the database, some checks are performed whenever the `run_checks.py` script is ran (the checks are performed on step 4 as described at the top of this document).


The checks' queries are all defined under `expectations/query_unit_tests.yml`, the purpose of each query is to pull the exact amount of a certain type of contract or tender and perform that same query on the dataframe obtained when `run_queries` is ran and pulls the data.


The raw queries used to obtain data from the database to compare to the dataframes is defined under each unit test's key with the key `raw_query`. The queries performed by `sqldf` (a function from the `pandasql` python package) are defined under the `sqldf_query` key.


As a note, considering that the initial queries have already been filtered by `import_pending = 'false'` and by a date (through the `date_from_where_to_check` key in the `config.yml`), these filters are not required to be passed to the `sqldf_query` key.


The `query_unit_checks` function basically obtains the count for each 'kind' of contract or tender (i.e. only minor contracts, tenders with no proposals, tenders with proposals, etc.) from the main database, then it obtains the counts for the same 'kinds' of contracts/tenders in the queried datasets by the `run_queries` function, compares them to see if they're the same and computes the difference in amounts to then present it neatly on a table. These values are used to determine if the process should continue or stop, as having differing counts would mean either missing contracts/tenders or exposing ourselves to the possibility of misflagging contracts/tenders.


The final table for the unit tests looks like this:

| table | unit_test | count | comparison_with_queries | difference | does_it_match |
|----|:---|---:|---:|---:|:---|
| tenders | total_tenders | 2901 | 2901 | 0 | True |
| contracts | total_contracts | 7688 | 7688 | 0 | True |
| contracts | minor_contracts | 4616 | 4616 | 0 | True |
| contracts | non_minor_contracts | 3067 | 3067 | 0 | True |
| tenders | tender_with_no_contracts | 391 | 391 | 0 | True |
| tenders | tender_with_contract | 2135 | 2135 | 0 | True |
| contracts | contract_with_no_tender | 4700 | 4700 | 0 | True |
| contracts | contract_with_tender | 2988 | 2988 | 0 | True |
| tenders | tender_with_lots | 863 | 863 | 0 | True |
| tenders | tender_without_lots | 2038 | 2038 | 0 | True |

This table is NOT pushed to the database, it's only used for diagnostic purposes during the run.


## **Totals**

Generating the totals table is the 7th step as described previously.


The checks for totals perform the following query and counts the amount of unique `id` present in the table. The tables checked are defined in the `config.yml` file, in the `totals_tables` key as a list, currently the tables checked are:

* tenders
* contracts
* private_entities
* public_entities
* documents

  \

The totals checks are performed by the function `generate_totals_table`, which essentially runs the sql query:

```sql
select count(distinct id) from {table}
```

Where `{table}` corresponds to each one of the tables in the previous list.

### **Result**

The final table generated would look like this (example counts are not real):

| date | table | count |
|:---|:---|---:|
| 2021-10-06 19:02:25.919312 | contracts | 178679 |
| 2021-10-06 19:02:25.919312 | tenders | 51284 |
| 2021-10-06 19:02:25.919312 | public_entities | 93790 |
| 2021-10-06 19:02:25.919312 | private_entities | 195516 |
| 2021-10-06 19:02:25.919312 | documents | 3750 |


## **Item checks over columns**

There are two main steps: running the expectations themselves (and saving their results), and constructing a dataframe where the IDs for elements which failed the checks are saved. These are the 8th and 9th steps described at the top of this document.

### **Running the expectations**

The `run_checks` function runs through the keys (table names) and expectations dictionary of the main `expectations` dict, which is constructed from the yaml files in the expectations directory.

Whenever a *key* is referenced, it refers to the keys in each yml config file for each table.

#### **Methodology (steps)**

* If either/or preprocessing/filtering is required:
  * Run any preprocessing steps described in the *preproc* key.
  * Filter the DataFrame if there's a filter to be applied, as described in the *filtered_df* key.
* The expectation is applied on the DataFrame, as described in the *expectations* key.
* The result is saved in the `expectations` dictionary's `result` key.

After this step, the table is generated.


### **Generating the table**

When the expectations have all ran, the results are neatly arranged on a table which will then be pushed to the SQL DB. The resulting table looks like this:

| date | item_id | table | check |
|:---|---:|:---|:---|
| 2021-10-06 | 1231134 | tenders | budget |
| 2021-10-06 | 1231614 | tenders | budget |
| 2021-10-06 | 1231638 | tenders | budget |
| 2021-10-06 | 1231642 | tenders | budget |
| 2021-10-06 | 1231694 | tenders | budget |

* *date* correspond to the date when the checks are made
* *item_id* corresponds to the unique `id` of the item on the table *table*
* *table* is the table where *item_id* failed the check *check*
* *check* is the check performed


## **Console parameter override when running** `run_checks.py` **script**

Whenever running the `run_checks.py` script through console, if we want to override a parameter from `config.yml`,  we can do so specifying the corresponding flag as follows:


```bash
python run_checks.py --date 2021-10-10
```


At the moment the following parameters from the `config.yml` can be overridden:


* `date_from_where_to_check` can be overridden by the flags `-d` and `--date`
* `db_insert_chunksize`  can be overridden by the flags `-cz` and `--chunksize`
* `remove_successful_items_from_db` can be overridden by the flags `-rs` and `--remove`
* `expectations_tables` can be overridden by the flags `-t` and `--table`


Each one has input validation:


* **dates** (`-d` and `--date`) have to be of the form `YYYY-MM-DD` and of length 10 (10 characters)
* **chunksize** (`-cz` and `--chunksize`) has to be between 100 and 1e5, inputs will be converted to `int`
* **remove** (`-rs` and `--remove`) has to be True or False, any input will be converted to `bool`, so inputting *anything* that isn't `0` or `False` explicitly will be converted to True, which is the default anyway
* **tables** (`-t` and `--table`) have to be a string with the name of a table in the database, if the table is NOT present in the database, an error will be returned


To see the help, you can use the `--help` and `-h` flags, which will return the information related to the possible parameters can be overridden and their current value in the `config.yml` :


```
Override GE checks arguments

optional arguments:
  -h, --help            show this help message and exit
  -d DATE, --date DATE  Date override for config.yml parameter "date_from_where_to_check".
                        *Currently configured to: 2021-11-11
  -cz CHUNKSIZE, --chunksize CHUNKSIZE
                        Chunksize override for config.yml parameter "db_insert_chunksize".
                        *Currently configured to: 1000
  -rs REMOVE, --remove REMOVE
                        Sucessful item (that had previously failed) removal override for
                        config.yml parameter "remove_successful_items_from_db".
                        *Currently configured to: True
  -t TABLE, --table TABLE
                        Table override for config.yml parameter "expectations_tables".
                        Only accepts a single table name
                        *Currently configured to: ['contracts', 'tenders', 'public_entities', 'private_entities']
```


## **Logs**

All logs for the checks are saved to the `/log/great_expectations.log` file in the gobierto-contratos repo inside the server. Whenever an information message or error is printed in console, it is also saved to the log file.


Within the code, any logs are written into the log file using the `logging` python library, a powerful library for creating/writing to logs. All instances of logging within the code are called with functions like `logging.info()` for information logs, `logging.warning()` for warnings, etc… This is all documented on the [logging HOWTO](https://docs.python.org/3/howto/logging.html#logging-basic-tutorial) in the python docs.


## **Expectation counts**

In order to keep tabs on new items failing expectations and on old items being fixed when reprocessing or updates are performed on the data, we count how many items were detected per expectation and per table, which we then append to `great_expectations_item_expectations_counts`  on the database. Here’s how the table looks like:



| id | expectation | item_type | count | date |
|----|----|----|----|----|
| 160 | assignee_id_exists | Contract | 0 | 2022-01-20 |
| 161 | assignee_entity_id_exists | Contract | 98 | 2022-01-20 |
| 162 | contractor_id_exists | Contract | 0 | 2022-01-20 |
| 163 | contractor_entity_id_exists | Contract | 2899 | 2022-01-20 |
| 178 | budget_between_values | Tender | 1401 | 2022-01-20 |
| 179 | contractor_entity_id_exists | Tender | 3012 | 2022-01-20 |
| 180 | contractor_id_exists | Tender | 0 | 2022-01-20 |
| 191 | dir3_exists | PublicEntity | 0 | 2022-01-20 |
| 192 | nif_exists | PrivateEntity | 1098 | 2022-01-20 |


This table shows how many unique IDs were detected failing each expectation for each table (like `contracts` or `tenders` etc..) on a specific date.


Like the totals table, this check is performed every day and is appended to the database.
