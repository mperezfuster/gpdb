# postgresml

[PostgresML](https://postgresml.org) is a machine learning extension for PostgreSQL that enables you to perform training and inference on text and tabular data using SQL queries. With PostgresML, you can seamlessly integrate machine learning models into your VMware Greenplum database and harness the power of cutting-edge algorithms to process data efficiently.

The `postgresml` module provides PostgresML functions for using tens of thousands of pre-trained open source AI/machine learning models in VMware Greenplum. The models are provided by the [Hugging Face AI data science platform](https://huggingface.co/). 

## <a id="prereqs"></a>Before Registering the `postgresml` Module

Before registering the `postgresml` module, you must install the Data Science bundle for Python3.9, add the `pgml` library to the set of libraries the VMware Greenplum server loads at startup, and set the Python virtual environment:

1. Install the Data Science bundle for Python 3.9.

    For example:

    ```
    gppkg install DataSciencePython3.9-x.x.x-gp7-el8_x86_64.gppkg 
    ```

    where x.x.x is the version string.

2. Add the `postgresml` library to preload when the VMware Greenplum server starts, using the `shared_preload_libraries` server configuration parameter and then restart the cluster.

    For example:

    ```
    gpconfig -c shared_preload_libraries -v 'xxx, pgml' 
    ```

    where xxx represents any existing preloaded libraries.

3. Set the Python virtual environment. 

    At the session level:

    ```
    SET pgml.venv='$GPHOME/ext/DataSciencePython3.9';
    ```

    To last beyond a session:

    ```
    gpconfig -c pgml.venv -v '$GPHOME/ext/DataSciencePython3.9'
    gpstop -u
    ```

>**Note**
>If this yields the error message, `[CRITICAL]:-not a valid GUC: pgml.venv` it means you failed to run Step 2, including restarting the cluster, beforehand.

Proceed to the next section to register the `postgresml` module.

## <a id="install_register"></a>Registering the Module

The `postgresml` module is installed when you install Greenplum Database. Before you can use any of the data types, functions, or operators defined in the module, you must register the `postgresml` extension in each database in which you want to use the objects:

```
CREATE EXTENSION pgml;
```

Refer to [Installing Additional Supplied Modules](../../install_guide/install_modules.html) for more information.

## <a id="topic_upgrading"></a>Upgrading the Module

The `PostgresML` module is installed when you install or upgrade Greenplum Database. A previous version of the extension will continue to work in existing databases after you upgrade Greenplum. To upgrade to the most recent version of the extension, you must:

```
ALTER EXTENSION pgml UPDATE TO '2.7.13+greenplum.1.0.0';
```

in **every** database in which you registered/use the extension.

## <a id="UDF_summary"></a>User-Defined Functions

The `postgresml` extension currently supports just a subset of all of the user-defined functions provided by PostgresML. They are these three:

- `pgml.load_dataset()`: Loads a dataset into tables in VMware Greenplum using the `INSERT` SQL command. Read more about loading data [here](https://postgresml.org/docs/guides/machine-learning/natural-language-processing/).
- `pgml.embed()` - Generates an embedding for the dataset. Read more about PostgresML embeddings [here](https://postgresml.org/docs/guides/machine-learning/natural-language-processing/embeddings). 
- `pgml.transform()`: Applies a pre-trained transformer to process data. Read more about PostgresML pre-trained models [here](https://postgresml.org/docs/guides/machine-learning/natural-language-processing/).
- `pgml.train()`: Handles different training tasks which are configurable with the function parameters.

VMware anticipates adding support for the additional PostgresML functions in future releases. 

### <a id="pgml_load_dataset"></a>pgml.load_dataset()

#### Syntax

```
pgml.load_dataset( 
	source TEXT,
	subset TEXT,
	limit bigint,
	kwargs JSONB
)
```

where:

`source` is the name of the data source.
`subset` is a subset of the data source. The default is `NULL`. 
`limit` is a user-defined limit on the number of imported rows. The default is `NULL`.
`kwargs` is a a JSONB object containing optional arguments. The default is an empty object (`{}`).


### <a id="pgml_embed"></a>pgml.embed()

#### Syntax

```
pgml.embed(
    transformer TEXT, 
    inputs TEXT or TEXT[],
    kwargs JSONB
)
```

where: 

- `transformer` is the huggingface sentence-transformer name.
- `inputs` are the text(s) to embed. It can be a single text string or an array of texts.
- `kwargs` is a JSONB object containing optional arguments.

### <a id="pgml_transform"></a>pgml.transform()

#### Syntax

```
pgml.transform(
    task TEXT or JSONB, 
    call JSONB,
    inputs TEXT[] or BYTEA[]
)
```

where: 

- `task` is the task name passed as either a simple text string or, for a more complex task setup, a JSONB object containing a full pipeline and initializer arguments.
- `call` is a JSONB object containing call arguments passed alongside the `inputs` values.
- `inputs` is a `TEXT[]` or `BYTEA[]` array containing inputs for inference. 

>**Note**
>You must explicitly specify a model when calling `pgml.transform()`; default models are not yet supported.

### <a id="pgml_train"></a>pgml.train()

#### Syntax

```
pgml.train(
    project_name TEXT,
    task TEXT DEFAULT NULL,
    relation_name TEXT DEFAULT NULL,
    y_column_name TEXT DEFAULT NULL,
    algorithm TEXT DEFAULT 'linear',
    hyperparams JSONB DEFAULT '{}'::JSONB,
    search TEXT DEFAULT NULL,
    search_params JSONB DEFAULT '{}'::JSONB,
    search_args JSONB DEFAULT '{}'::JSONB,
    test_size REAL DEFAULT 0.25,
    test_sampling TEXT DEFAULT 'random',
    preprocess JSONB DEFAULT '{}'::JSONB
)
```

where

- `project_name` is an easily recognizable identifier to organize your work.
- `task` is the objective of the experiment: `regression`, `classification` or `cluster`.
- `relation_name` is the Postgres table or view where the training data is stored or defined.
- `y_column_name` is the name of the label column in the training table.
- `algorithm` is the algorithm to train on the dataset, see the task specific pages for available algorithms: [regression](#pgml_train_algorithms_regression), [classification](#pgml_train_algorithms_classification), [clustering](#pgml_train_algorithms_clustering).
- `hyperparams` are the hyperparameters to pass to the algorithm for training, JSON formatted.
- `search` indicates, if set, whether or not PostgresML performs a hyperparameter search to find the best hyperparameters for the algorithm. See [Hyperparameter Search](#pgml_train_hyperparameter) for details.
- `search_params` are the search parameters used in the `hyperparameter` search, using the scikit-learn notation, JSON formatted.
- `search_args` are the configuration parameters for the `search`, JSON formatted. Currently only `n_iter` is supported for `random` search.
- `test_size` is the fraction of the dataset to use for the test set and algorithm validation.
- `test_sampling` is the algorithm used to fetch test data from the dataset: `random`, `first`, or `last`.
- `preprocess` are the preprocessing steps to impute NULLS, encode categoricals and scale inputs. See [Data Pre-Processing](#pgml_train_datapreproc) for details.

#### <a id="pgml_train_algorithms"></a>Algorithms

##### <a id="pgml_train_algorithms_regression"></a>Regression

PostgresML currently supports regression algorithms from [scikit-learn](https://scikit-learn.org/), [XGBoost](https://xgboost.readthedocs.io/), [LightGBM](https://lightgbm.readthedocs.io/) and [Catboost](https://catboost.ai/).

##### <a id="pgml_train_algorithms_classification"></a>Classification

This technique assigns new observations to categorical labels or classes based on a model built from labeled training data.

##### <a id="pgml_train_algorithms_clustering"></a>Clustering

You can train models using `pgml.train` on unlabeled data to identify groups within the data. To build clusters on a given dataset, you can use the table or a view. Since clustering is an unsupervised algorithm, you do not need a column that represents a label as one of the inputs to `pgml.train`.

#### <a id="pgml_train_hyperparameter"></a>Hyperparameter Search

You can further refine the models by using hyperparameter search and cross validation. PostgresML currently supports `random` and `grid` search algorithms, and `k-fold` cross validation.

#### <a id="pgml_train_datapreproc"></a>Data Pre-Processing

The training function also provides the option to pre-process data with the `preprocess` parameter. You can configure the pre-processors on a per-column basis for the training data set. There are currently three types of preprocessing available, for both categorical and quantitative variables. Below is a brief example for training data to learn a model of whether we should carry an umbrella or not.

Preprocessing steps are saved after training, and repeated identically for future calls to `pgml.predict()`.

## <a id="Example"></a>Example

The following example:

1. Downloads a dataset from the internet and creates a table to contain the data.
2. Generates an embedding for the text.
3. Downloads and runs pre-trained models.

```
# Download the dataset from the internet and create table for it

SELECT pgml.load_dataset('tweet_eval', 'sentiment'); 

# Generate an embedding for the text 

SELECT pgml.embed('distilbert-base-uncased', 'Star Wars christmas special is on Disney')::vector AS embedding; 
--------------------------------------------------------------------------------------------- 

SELECT text, pgml.embed('distilbert-base-uncased', text) 

FROM pgml.tweet_eval limit 5; 

--------------------------------------------------------------------------------------------- 

CREATE TABLE tweet_embeddings AS 

SELECT text, pgml.embed('distilbert-base-uncased', text) AS embedding 

FROM pgml.tweet_eval limit 5; 

# Download and run pre-trained models

SELECT pgml.transform( 
    'translation_en_to_fr', 
    inputs => ARRAY[ 
        'Welcome to the future!', 
        'Where have you been all this time?' 
    ] 
) AS french; 
```
