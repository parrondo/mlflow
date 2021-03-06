.. _tracking:

MLflow Tracking
===============

The MLflow Tracking component is an API and UI for logging parameters, code versions, metrics, and output files when running your machine learning code and for later visualizing the results. MLflow Tracking lets you log and query experiments using both Python and REST APIs.

.. contents:: Table of Contents
  :local:
  :depth: 1

Concepts
--------

MLflow Tracking is organized around the concept of *runs*, which are executions of some piece of
data science code. Each run records the following information:

Code Version
    Git commit used to execute the run, if it was executed from an :ref:`MLflow Project <projects>`.

Start & End Time
    Start and end time of the run

Source
    Name of the file executed to launch the run, or the project name and entry point for the run
    if the run was executed from an :ref:`MLflow Project <projects>`.

Parameters
    Key-value input parameters of your choice. Both keys and values are strings.

Metrics
    Key-value metrics where the value is numeric. Each metric can be updated throughout the
    course of the run (for example, to track how your model's loss function is converging), and
    MLflow will record and let you visualize the metric's full history.

Artifacts
    Output files in any format. For example, you can record images (for example, PNGs), models
    (for example, a pickled scikit-learn model) or even data files (for example, a
    `Parquet <https://parquet.apache.org/>`_ file) as artifacts.

Runs can be recorded from anywhere you run your code through MLflow's Python and REST APIs: for
example, you can record them in a standalone program, on a remote cloud machine, or in an
interactive notebook. If you record runs in an :ref:`MLflow Project <projects>`, MLflow
remembers the project URI and source version.

Finally, runs can optionally be organized into *experiments*, which group together runs for a
specific task. You can create an experiment via the ``mlflow experiments`` CLI, with
:py:func:`mlflow.create_experiment`, or via the corresponding REST parameters. The MLflow API and
UI let you create and search for experiments.

Once your runs have been recorded, you can query them using the :ref:`tracking_ui` or the MLflow
API.

Where Runs Get Recorded
-----------------------

MLflow runs can be recorded either locally in files or remotely to a tracking server.
By default, the MLflow Python API logs runs to files in an ``mlruns`` directory wherever you
ran your program. You can then run ``mlflow ui`` to see the logged runs. Set the
``MLFLOW_TRACKING_URI`` environment variable to a server's URI or call
:py:func:`mlflow.set_tracking_uri` to log runs remotely.

There are a different kinds of remote tracking URIs:

- Local file path (specified as ``file:/my/local/dir``), where data is just directly stored locally.
- HTTP server (specified as ``https://my-server:5000``), which is a server hosting :ref:`your own tracking server <tracking_server>`.
- Databricks workspace (specified as ``databricks``, or a specific Databricks CLI profile as ``databricks://profileName``. For more information on configuring a Databricks CLI, see `here <https://github.com/databricks/databricks-cli>`_. This only works for workspaces for which the Databricks MLflow Tracking Server is enabled; please contact Databricks if interested.

Logging Data to Runs
--------------------

You can log data to runs using either the MLflow Python or REST API. This section
shows the Python API.

.. _basic_logging_functions:

Basic Logging Functions
^^^^^^^^^^^^^^^^^^^^^^^

:py:func:`mlflow.set_tracking_uri` connects to a tracking URI. You can also set the
``MLFLOW_TRACKING_URI`` environment variable to have MLflow find a URI from there. In both cases,
the URI can either be a HTTP/HTTPS URI for a remote server, or a local path to log data to a
directory. The URI defaults to ``mlruns``.

:py:func:`mlflow.get_tracking_uri` returns the current tracking URI.

:py:func:`mlflow.create_experiment` creates a new experiment and returns its ID. Runs can be
launched under the experiment by passing the experiment ID to ``mlflow.start_run``.

:py:func:`mlflow.start_run` returns the currently active run (if one exists), or starts a new run
and returns a :py:class:`mlflow.tracking.ActiveRun` object usable as a context manager for the
current run. You do not need to call ``start_run`` explicitly: calling one of the logging functions
with no active run will automatically start a new one.

:py:func:`mlflow.end_run` ends the currently active run, if any, taking an optional run status.

:py:func:`mlflow.active_run` returns a :py:class:`mlflow.tracking.Run` object corresponding to the
currently active run, if any.

:py:func:`mlflow.log_param` logs a key-value parameter in the currently active run. The keys and
values are both strings.

:py:func:`mlflow.log_metric` logs a key-value metric. The value must always be a number. MLflow will
remember the history of values for each metric.

:py:func:`mlflow.log_artifact` logs a local file as an artifact, optionally taking an
``artifact_path`` to place it in within the run's artifact URI. Run artifacts can be organized into
directories, so you can place the artifact in a directory this way.

:py:func:`mlflow.log_artifacts` logs all the files in a given directory as artifacts, again taking
an optional ``artifact_path``.

:py:func:`mlflow.get_artifact_uri` returns the URI that artifacts from the current run should be
logged to.


Launching Multiple Runs in One Program
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Sometimes you want to execute multiple MLflow runs in the same program: for example, maybe you are
performing a hyperparameter search locally or your experiments are just very fast to run. This is
easy to do because the ``ActiveRun`` object returned by :py:func:`mlflow.start_run` is a Python
`context manager <https://docs.python.org/2.5/whatsnew/pep-343.html>`_. You can "scope" each run to
just one block of code as follows:

.. code:: python

   with mlflow.start_run():
       mlflow.log_param("x", 1)
       mlflow.log_metric("y", 2)
       ...

The run remains open throughout the ``with`` statement, and is automatically closed when the
statement exits, even if it exits due to an exception.

Organizing Runs in Experiments
------------------------------

MLflow allows you to group runs under experiments, which can be useful for comparing runs intended
to tackle a particular task. You can create experiments via the CLI (``mlflow experiments``) or via
the :py:func:`create_experiment` Python API. You can pass the experiment ID for a individual run
via the CLI (for example, ``mlflow run ... --experiment-id [ID]``) or via the ``MLFLOW_EXPERIMENT_ID``
environment variable.

.. code:: bash

    # Prints "created an experiment with ID <id>
    mlflow experiments create fraud-detection
    # Set the ID via environment variables
    export MLFLOW_EXPERIMENT_ID=<id>

.. code:: python

    # Launch a run. The experiment ID is inferred from the MLFLOW_EXPERIMENT_ID environment
    # variable, or from the --experiment-id parameter passed to the MLflow CLI (the latter
    # taking precedence)
    with mlflow.start_run():
        mlflow.log_param("a", 1)
        mlflow.log_metric("b", 2)

Managing Experiments and Runs with the Tracking Service API
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

MLflow provides a more detailed Tracking Service API for managing experiments and runs directly, which is available in the :doc:`mlflow.tracking<python_api/mlflow.tracking>` package. This makes it possible to query data about past runs, log additional information about them, create experiments and more.

Example usage:

.. code:: python

    from  mlflow.tracking import get_service
    service = get_service()
    experiments = service.list_experiments() # returns a list of mlflow.entities.Experiment
    run = service.create_run(experiments[0].experiment_id) # returns mlflow.entities.Run
    service.log_param(run.info.run_uuid, "hello", "world")
    service.set_terminated(run.info.run_uuid)

.. _tracking_ui:

Tracking UI
-----------

The Tracking UI lets you visualize, search and compare runs, as well as download run artifacts or
metadata for analysis in other tools. If you have been logging runs to a local ``mlruns`` directory,
run ``mlflow ui`` in the directory above it, and it will load the corresponding runs.
Alternatively, the :ref:`MLflow Server <tracking_server>` serves the same UI, and enables remote storage of run artifacts.

The UI contains the following key features:

* Experiment-based run listing and comparison
* Searching for runs by parameter or metric value
* Visualizing run metrics
* Downloading run results

.. _tracking_query_api:

Querying Runs Programmatically
------------------------------

All of the functions in the Tracking UI can be accessed programmatically through the
:py:mod:`mlflow.tracking` module and the REST API. This makes it easy to do several
common tasks:

* Query and compare runs using any data analysis tool of your choice, for example, **pandas**.
* Determine the artifact URI for a run to feed some of its artifacts into a new run when executing
  a workflow.
* Load artifacts from past runs as :ref:`models`.
* Run automated parameter search algorithms, where you query the metrics from various runs to
  submit new ones.

.. _tracking_server:

Running a Tracking Server
-------------------------

The MLflow tracking server launched via ``mlflow server`` also hosts REST APIs for tracking runs,
writing data to the local filesystem. You can specify a tracking server URI
via the ``MLFLOW_TRACKING_URI`` environment variable and MLflow's tracking APIs automatically
communicate with the tracking server at that URI to create/get run information, log metrics, and so on.

An example configuration for a server is as follows:

.. code:: bash

    mlflow server \
        --file-store /mnt/persistent-disk \
        --default-artifact-root s3://my-mlflow-bucket/ \
        --host 0.0.0.0

Storage
^^^^^^^

The tracking server has two properties related to how data is stored: File Store and Artifact Store.

The **File Store** (exposed via ``--file-store``) is where the *server* stores run and experiment metadata.
It defaults to the local ``./mlruns`` directory (same as when running ``mlflow run`` locally), but when
running a server, make sure that this points to a persistent (that is, non-ephemeral) file system location.

The **Artifact Store** is a location suitable for large data (such as an S3 bucket or shared NFS file system)
where *clients* log their artifact output (for example, models). The Artifact Store is a property
of an experiment, but the ``--default-artifact-root`` flag sets the artifact root URI for
newly-created experiments that do not specify one. Once you create an experiment, the ``--default-artifact-root`` is no longer relevant to it.

To allow the clients and server to access the artifact location, you should configure your cloud
provider credentials as normal. For example, for S3, you can set the ``AWS_ACCESS_KEY_ID``
and ``AWS_SECRET_ACCESS_KEY`` environment variables, use an IAM role, or configure a default
profile in ``~/.aws/credentials``. See `Set up AWS Credentials and Region for Development <https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/setup-credentials.html>`_ for more info.

.. important::

  If you do not specify a ``--default-artifact-root`` or an artifact URI when creating the experiment (for example, ``mlflow experiments create --artifact-root s3://<my-bucket>``), then the artifact root will be a path inside the File Store. Typically this is not an appropriate location, as the client and server will probably be referring to different physical locations (that is, the same path on different disks).

Supported Artifact Stores
^^^^^^^^^^^^^^^^^^^^^^^^^
In addition to local file paths, MLflow supports the following storage systems as artifact stores: 
Amazon S3, Azure Blob Storage, and Google Cloud Storage.

Amazon S3
~~~~~~~~~
Specify a URI of the form ``s3://<bucket>/<path>`` to store artifacts in S3. MLflow obtains
credentials to access S3 from your machine's IAM role, a profile in ``~/.aws/credentials``, or
the environment variables ``AWS_ACCESS_KEY_ID`` and ``AWS_SECRET_ACCESS_KEY`` depending on which of
these are available. See
`Set up AWS Credentials and Region for Development <https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/setup-credentials.html>`_ for more information on how to set credentials.

To store artifacts in a custom endpoint, please set the ``MLFLOW_S3_ENDPOINT_URL`` to your endpoint's URL. For example, if you have a Minio server at 1.2.3.4 on port 9000:

.. code:: bash

  export MLFLOW_S3_ENDPOINT_URL=http://1.2.3.4:9000

Azure Blob Storage
~~~~~~~~~~~~~~~~~~
Specify a URI of the form ``wasbs://<container>@<storage-account>.blob.core.windows.net/<path>`` to store
artifacts in Azure Blob Storage. MLflow looks for your Azure Storage access credentials in the
``AZURE_STORAGE_CONNECTION_STRING`` or ``AZURE_STORAGE_ACCESS_KEY`` environment variables (preferring
a connection string if one is set), so you will need to set one of these variables on both your client
application and your MLflow tracking server. Finally, you will need to ``pip install azure-storage``
separately (on both your client and the server) to access Azure Blob Storage; MLflow does not declare
a dependency on this package by default.

Google Cloud Storage
~~~~~~~~~~~~~~~~~~~~
Specify a URI of the form ``gs://<bucket>/<path>`` to store artifacts in Google Cloud Storage.
You should configure credentials for accessing the GCS container on the client and server as described
in the `GCS documentation <https://google-cloud.readthedocs.io/en/latest/core/auth.html>`_.
Finally, you will need to ``pip install google-cloud-storage`` (on both your client and the server)
to access Google Cloud Storage; MLflow does not declare a dependency on this package by default.

SFTP server
~~~~~~~~~~~
Specify a URI of the form sftp://user@host/path/to/directory to store artifacts in an SFTP server. You should configure the client to be able to login to the SFTP server without a password over SSH (e.g. public key, identityfile in ssh_config...).

In principle the following format ``sftp://pass:user@host/`` is supported for logging in. But for safety reasons this is not recommended.

When using this store, ``pysftp`` has to be installed on both client and server. Simply ``pip install pysftp`` will install the required package.

Networking
^^^^^^^^^^

The ``--host`` option exposes the service on all interfaces. If running a server in production, we
would recommend not exposing the built-in server broadly (as it is unauthenticated and unencrypted),
and instead putting it behind a reverse proxy like NGINX or Apache httpd, or connecting over VPN.
Additionally, you should ensure that the ``--file-store`` (which defaults to the ``./mlruns`` directory)
points to a persistent (non-ephemeral) disk.

Connecting to a Remote Server
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Once you have a server running, set ``MLFLOW_TRACKING_URI`` to the server's URI, along
with its scheme and port (for example, ``http://10.0.0.1:5000``). Then you can use ``mlflow`` as normal:

.. code:: python

    import mlflow
    with mlflow.start_run():
        mlflow.log_metric("a", 1)

The ``mlflow.start_run`` and ``mlflow.log_metric`` calls make API requests to your remote
tracking server.
