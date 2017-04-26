### Disclaimer

This is not an official Google product.

# dsub: simple batch jobs with Docker

## Overview

dsub is a command-line tool that makes it easy to submit and run batch scripts
in the cloud. dsub uses Docker, which makes it easy to package up portable
code that people can run anywhere Docker is supported.

The dsub user experience is modeled after traditional high-performance
computing job schedulers like Grid Engine and Slurm. You write a script and
then submit it to a job scheduler from a shell prompt on your local machine.

For now, dsub supports Google Cloud as the backend batch job runner. With help
from the community, we'd like to add other backends, such as a local runner,
Grid Engine, Slurm, Amazon Batch, and Azure Batch.

If others find dsub useful, our hope is to contribute dsub to an open-source
foundation for use by the wider batch computing community.

## Getting started

1.  Clone this repository.

        git clone https://github.com/googlegenomics/dsub
        cd dsub

1.  Setup a Python virtualenv (optional but strongly recommended).

        virtualenv dsub_libs
        source dsub_libs/bin/activate

1.  Install dependent libraries.

        pip install --upgrade oauth2client==1.5.2 google-api-python-client python-dateutil pytz tabulate

1.  Verify the installation by running:

        ./dsub --help

1.   (Optional) [Install Docker](https://docs.docker.com/engine/installation/).

     This is necessary only if you're going to create your own Docker images.

### Getting started on Google Cloud

1.  Sign up for a Google Cloud Platform account and
    [create a project](https://console.cloud.google.com/project?).

1.  [Enable the APIs](https://console.cloud.google.com/flows/enableapi?apiid=genomics,storage_component,compute_component&redirect=https://console.cloud.google.com).

1.  [Install the Google Cloud SDK](https://cloud.google.com/sdk/) and run

        gcloud init

    This will set up your default project and grant credentials to the Google
    Cloud SDK. Now provide [credentials](https://developers.google.com/identity/protocols/application-default-credentials)
    so dsub can call Google APIs:

        gcloud application-default login

1.  Create a [Google Cloud Storage](https://cloud.google.com/storage) bucket.

    The dsub logs and output files will be written to a bucket. Create a
    bucket using the [storage browser](https://cloud.google.com/storage/browser?project=)
    or run the command-line utility [gsutil](https://cloud.google.com/storage/docs/gsutil), included in
    the Cloud SDK.

        gsutil mb gs://my-bucket

    Change `my-bucket` to a unique name that follows the
    [bucket-naming conventions](https://cloud.google.com/storage/docs/bucket-naming).

    (By default, the bucket will be in the US, but you can change or
    refine the [location](https://cloud.google.com/storage/docs/bucket-locations) setting with the
    `-l` option.)

## Running a job

Here's the simplest example:

    dsub \
        --project my-cloud-project \
        --logging gs://my-bucket/logs \
        --command 'echo hello'

Change `my-cloud-project` to your Google Cloud project, and `my-bucket` to
the bucket you created above.

After running dsub, the output will be a server-generated job id.
The output of the script command will be written to the logging folder.

The following sections show how to run more complex jobs.

### Defining what code to run

You can provide a shell command directly in the dsub command-line, as in the
hello example above.

You can also save your script to a file, like `hello.sh`. Then you can run:

    dsub \
        --project my-cloud-project \
        --logging gs://my-bucket/logs \
        --script hello.sh

If your script has dependencies that are not stored in your Docker image,
you can transfer them to the local disk. See the instructions below for
working with input and output files and folders.

### Selecting a Docker image

By default, dsub uses a stock Ubuntu image. You can change the image
by passing the `--image` flag.

    dsub \
        --project my-cloud-project \
        --logging gs://my-bucket/logs \
        --image ubuntu:16.04 \
        --script hello.sh

### Passing parameters to your script

You can pass environment variables to your script using the `--env` flag.

    dsub \
        --project my-cloud-project \
        --logging gs://my-bucket/logs \
        --env MESSAGE=hello \
        --command 'echo ${MESSAGE}'

The environment variable `MESSAGE` will be assigned the value `hello` when
your Docker container runs.

Your script or command can reference the variable like any other Linux
environment variable, as `${MESSAGE}`.

**Be sure to enclose your command string in single quotes and not double
quotes. If you use double quotes, the command will be expanded in your local
shell before being passed to dsub. For more information on using the
`--command` flag, see [Scripts, Commands, and Docker](docs/code.md)**

To set multiple environment variables, you can repeat the flag:

    --env VAR1=value1 \
    --env VAR2=value2

You can also set multiple variables, space-delimited, with a single flag:

    --env VAR1=value1 VAR2=value2

### Working with input and output files and folders

dsub mimics the behavior of a shared file system using cloud storage
bucket paths for input and output files and folders. You specify
the cloud storage bucket path. Paths can be:

* file paths like `gs://my-bucket/my-file`
* folder paths like `gs://my-bucket/my-folder`
* wildcard paths like `gs://my-bucket/my-folder/*`

See the [inputs and outputs](docs/input_output.md) documentation for more details.

### Transferring input files to a Google Cloud Storage bucket.

If your script expects to read local input files that are not already
contained within your Docker image, the files must be available in Google
Cloud Storage.

If your script has dependent files, you can make them available to your script
by:

 * Building a private Docker image with the dependent files and publishing the
   image to a public site, or privately to Google Container Registry
 * Uploading the files to Google Cloud Storage

To upload the files to Google Cloud Storage, you can use the
[storage browser](https://console.cloud.google.com/storage/browser?project=) or
[gsutil](https://cloud.google.com/storage/docs/gsutil). You can also run on data
that’s public or shared with your service account, an email address that you
can find in the [Google Cloud Console](https://console.cloud.google.com).

#### Files

To specify input and output files, use the `--input` and `--output` flags:

    dsub \
        --project my-cloud-project \
        --logging gs://my-bucket/logs \
        --input INPUT_FILE=gs://my-bucket/my-input-file \
        --output OUTPUT_FILE=gs://my-bucket/my-output-file \
        --command 'cat ${INPUT_FILE} > ${OUTPUT_FILE}'

The input file will be copied from `gs://my-bucket/my-input-file` to a local
path given by the environment variable `${INPUT_FILE}`. Inside your script, you
can reference the local file path using the environment variable.

The output file will be written to local disk at the location given by
`${OUTPUT_FILE}`. Inside your script, you can reference the local file path
using the environment variable. After the script completes, the output file
will be copied to the bucket path `gs://my-bucket/my-output-file`.

#### Folders

To copy folders rather than files, use the `--input-recursive` or
`output-recursive` flags:

    dsub \
        --project my-cloud-project \
        --logging gs://my-bucket/logs \
        --input-recursive FOLDER=gs://my-bucket/my-folder \
        --command 'find ${FOLDER} -name "foo*"'

### Setting resource requirements

By default, dsub launches a VM with a single CPU core, a default number of
GB of memory (3.75 GB on Google Compute Engine), and a default disk size
(200 GB).

To change the minimum RAM, use the `--min-ram` flag.

To change the minimum number of CPU cores, use the `--min-cores` flag.

To change the disk size, use the `--disk-size` flag.

Before you choose especially large or unusual values, be sure to check the
available VM instance types and maximum disk size. On Google Cloud, the
machine type will be selected from the best matching
[predefined machine types](https://cloud.google.com/compute/docs/machine-types#predefined_machine_types).

### Submitting a batch job

You can run a batch job consisting of an array of many tasks, rather than a
single task as in all of the examples so far. To run a batch of tasks, create
a tab-separated values (TSV) file containing the environment variables and
input and output parameters for multiple tasks.

The first line of the TSV file specifies the names and types of the
parameters. For example:

    --env SAMPLE_ID<tab>--input VCF_FILE<tab>--output OUTPUT_PATH

The first line also supports bare-word variables which are treated as
the names of environment variables. This example is equivalent to the previous:

    SAMPLE_ID<tab>--input VCF_FILE<tab>--output OUTPUT_PATH

Then pass this file to dsub using the `--table` parameter.

### Job control

It's possible to wait for a job to complete before starting another, see [job
control with dsub](docs/job_control.md).

### Viewing job status

The `dstat` command displays the status of jobs:

    ./dstat --project my-cloud-project

With no additional arguments, dstat will display a list of *running* jobs for
the current `USER`.

To display the status of a specific job, use the `--jobs` flag:

    ./dstat --project my-cloud-project --jobs job-id

For a batch job, the output will list all *running* tasks.

Each job submitted by dsub is given a set of metadata values that can be
used for job identification and job control. The metadata associated with
each job includes:

* `job-name`: defaults to the name of your script file or the first word of
  your script command; it can be explicitly set
  with the `--name` parameter.
* `user-id`: the `USER` environment variable value.
* `job-id`: takes the form `job-name--userid--timestamp` where the
  `job-name` is truncated at 10 characters and the `timestamp` is of the form
  `YYMMDD-HHMMSS-XX`, unique to hundredths of a second.
* `task-id`: if the job is a "table job", each task gets a sequential value
  of the form "task-*n*" where *n* is 1-based.

Metadata can be used to cancel a job or individual tasks within a batch job.

### Cancelling a job

The `ddel` command will cancel running jobs. To delete a running job:

    ./ddel --project my-cloud-project --jobs job-id

If the job is a batch job, all tasks will be canceled.

To cancel specific tasks, run:

    ./ddel \
        --project my-cloud-project \
        --jobs job-id \
        --tasks task-id1 task-id2

## What next?

* See the [examples](examples).
* See more documentation for:
  * [Scripts, Commands, and Docker](docs/code.md)
  * [Input and Output File Handling](docs/input_output.md)
  * [Job Control](docs/job_control.md)
  * [Checking Status and Troubleshooting Jobs](docs/troubleshooting.md)
