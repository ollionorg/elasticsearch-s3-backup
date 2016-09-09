# S3-Based Backup/Restore for Elasticsearch

## Updates: 9-Sep-2016
### support for AWS managed Elasticsearch service
- check out the `aws-ES-service` branch
- sign all ES HTTP requests with an appropriate IAM Role via [AWS4Auth](https://pypi.python.org/pypi/requests-aws4auth)... based on the [standard way to do this](https://elasticsearch-py.readthedocs.io/en/master/#running-on-aws-with-iam) in the Python-Elasticsearch Client

### support for Elastic.co Cloud's managed Elasticsearch service
- check out the `elastic.co` branch
- pass [Shield](https://www.elastic.co/products/shield) auth credentials via http basic auth

## Summary
The `es-s3-snapshot` utility can be used to backup and restore Elasticsearch (ES) indices from any ES node/cluster to another ES node/cluster.

Backups are stored as [ES snapshots](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html) in an [Amazon AWS S3](https://aws.amazon.com/s3/) bucket.

The utility uses Elasticsearch's in-built [Snapshot and Restore API](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html). The standard [Elasticsearch AWS Cloud Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/cloud-aws.html) permits use of an S3 bucket for the ES snapshot repository.

## Why Use S3 for Elasticsearch Snapshots

S3 works great as an ES snapshot/backup store:
* Durable backups: S3 has [high durability and availability](https://aws.amazon.com/s3/faqs/) which is important for backups in general.
* Backup size: S3 supports a maximum object size of 5TB which is more than sufficient for ES. Each ES snapshot is split into multiple small chunks which never hit the max object size. Hence S3 can be used to backup even huge indices.
* Speed: With filesystem-based snapshots, if not using a shared/distributed file system such as NFS, you probably have to transmit the snapshot files over to the target ES cluster. This can take a long time if the snapshots are large and if there are several files. With S3 functioning as a "shared" snapshot repository, this is less of an issue.


## Primary Use Case
You can take a snapshot of pre-specified ES indices from a **source** ES cluster and save the snapshot in S3. This is called `backup` mode.

This snapshot can then be used in `restore` mode to push the previously-S3-snapshotted data to a **destination** ES cluster. This is called `restore` mode.

All snapshots are kept in a pre-specified S3 bucket. The `es-s3-snapshot` utility does not perform file-system based snapshots.

## Prerequisites and Setup

The utility depends on `Python 2.7` and the standard [Elasticsearch Python client library](https://Elasticsearch-py.readthedocs.org/en/master/).

It also requires the Elasticsearch [AWS Cloud Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/cloud-aws.html) to be installed on **every node of both source and target ES clusters**, otherwise you will [see errors like this](https://discuss.elastic.co/t/unknown-repository-type-s3-when-creating-snapshot-in-2-x/35697/7).

The `es-s3-snapshot` utility has been tested on these platforms:
* Amazon Linux
* CentOS
* Redhat


### Install Elasticsearch AWS Cloud Plugin
The `AWS Cloud Plugin` for Elasticsearch needs to be installed on every node of both the source and target ES cluster and **you have to restart the ES service on each node after installing the plugin** on it.

You can install it via the ES plugin manager:

```sh
sudo bin/plugin install cloud-aws
```

For more details, see the [AWS Cloud Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/cloud-aws.html) instructions.

### Install Python 2.7 and pip

Both Python2.7 and `pip` (the Python package manager) are required to install the Elasticsearch Python client library which is needed to run the `es-s3-snapshot` utility.

These dependencies have to be installed on one node in your source ES cluster and one node in your destination ES cluster.

On RPM-based Linux distributions such as CentOS/RHEL/Amazon Linux, use `yum`, for example:

``` sh
yum install Python27 Python27-pip
```

On Debian-based Linux distributions such as Ubuntu, use `aptitude`, for example:

```sh
apt-get update
apt-get install python2.7 python-pip
```

### Install the Python Elasticsearch Client Library

Python's standard [Elasticsearch client library](https://elasticsearch-py.readthedocs.org/en/master/) enforces the following requirements between the version of the actual Elasticsearch Service running on a given host and the version of the Elasticsearch Python library.

The Python ES library is compatible with all Elasticsearch versions since `0.90.x` but you have to use a matching major version:
* For Elasticsearch `2.0` and later, use the major version `2 (2.x.y)` of the library.
* For Elasticsearch `1.0` and later, use the major version `1 (1.x.y)` of the library.
* For Elasticsearch `0.90.x`, use a version from `0.4.x` releases of the library.


Use `pip` (Python's package manager) to install the Python Elasticsearch Library - both on the **source** (`src`) ES cluster and the **destination** (`dest`) ES cluster.

If you are using Elasticsearch 1.x on your node/cluster:

```sh
pip install 'Elasticsearch>=1.0.0,<2.0.0'
```

If you are using Elasticsearch 2.x on your node/cluster:

```sh
pip install 'Elasticsearch>=2.0.0,<3.0.0'
```

This has to be done on one node in your source ES cluster and one node in your destination ES cluster.


## How To Use The `es-s3-snapshot` Utility

Now that your setup is complete, you can use the `es-s3-snapshot` utility in either `backup` mode or in `restore` mode.

You run it in `backup` mode on the source (`src` in the conf file) ES cluster where you want to take a snapshot of your ES indices.

You run it in `restore` mode on the target (`dest` in the conf file) ES cluster where you want to restore the previously-snapshotted ES indices.


### IAM Policy
Since the tool only has to write to a specific S3 bucket, it only needs permissions to that bucket, for example:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:*"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::YOUR_BUCKET_NAME",
                "arn:aws:s3:::YOUR_BUCKET_NAME/*"
            ]
        }
    ]
}
```

You can create an IAM user, attach the above IAM policy to this user and use the corresponding ACCESS and SECRET keys in the configuration below.


### Configuration Details
The `es-s3-snapshot.conf` file contains all the required runtime parameters for the utility.

Before running the `es-s3-snapshot` utility, ensure that you have entered valid values in the `es-s3-snapshot.conf` file. This is a critical and mandatory step.

For example, without setting the correct value of your AWS ACCESS and SECRET keys in the configuration file, the `es-s3-snapshot` utility will simply not work.

Please refer to `es-s3-snapshot.conf.SAMPLE` which contains representative values and comments that make the config self-explanatory:

```sh
[aws_api_keys]
# Enter your actual, working AWS API keys here:
aws_access_key = AKIAXXXYYYZZZAAABBB
aws_secret_key = OIAAAAABBBBBCCCCCDDDDDEEEE


[aws_s3_config]
aws_region = us-west-2
s3_bucket_name = my-s3-devops-bucket
s3_base_path = elasticsearch-backups  # the s3 'path' inside the bucket for ES snapshots


[elasticsearch_config]
es_repository_name = my_es_snapshots

# SOURCE nodes: BACKUP from here
es_src_seed1 = 10.40.11.180:9200

# if your ES source is a cluster instead of a single node,
# then uncomment the following lines and set the correct IP addresses:
#es_src_seed2 = a.b.c.d:9200
#es_src_seed3 = w.x.y.z:9200


# DESTINATION nodes: RESTORE to here
es_dest_seed1 = 10.40.101.198:9200
es_dest_seed2 = 10.40.101.224:9200
es_dest_seed3 = 10.40.101.55:9200

# comma-separated names of source indices to backup
index_names = my_index1,my_index2,my_index_3

# name of ES snapshot to create in S3
snapshot_name = snapshot_test_1
```



## Backup Mode

### Backup can be "safely" run on a running cluster
From the [official ES docs on snapshot+restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html):

> The index snapshot process is incremental. In the process of making the index snapshot Elasticsearch analyses the list of the index files that are already stored in the repository and copies only files that were created or changed since the last snapshot. That allows multiple snapshots to be preserved in the repository in a compact form. **Snapshotting process is executed in non-blocking fashion**. All indexing and searching operation can continue to be executed against the index that is being snapshotted. However, a snapshot represents the point-in-time view of the index at the moment when snapshot was created, so no records that were added to the index after the snapshot process was started will be present in the snapshot. The snapshot process starts immediately for the primary shards that has been started and are not relocating at the moment. Elasticsearch waits for relocation or initialization of shards to complete before snapshotting them.

> Besides creating a copy of each index the snapshot process can also store global cluster metadata, which includes persistent cluster settings and templates. The transient settings and registered snapshot repositories are not stored as part of the snapshot.

> Only one snapshot process can be executed in the cluster at any time. While snapshot of a particular shard is being created this shard cannot be moved to another node, which can interfere with rebalancing process and allocation filtering. Elasticsearch will only be able to move a shard to another node (according to the current allocation filtering settings and rebalancing algorithm) once the snapshot is finished.

### Take an S3 Snapshot of ES Indices
First ensure that you have entered valid values in the `es-s3-snapshot.conf` file. This is critical. See the previous section for details.

Once you have a correct `es-s3-snapshot.conf` file, simply run the utility in `backup` mode on a source ES node (or any other machine which can see port 9200 on a source ES node) like this:

```sh
python es-s3-snapshot.py -m backup
```

The backup operation can be performed on any node of the source cluster.

If all runs properly, you should see something like this in `/var/log/elasticsearch/elasticsearch.log`:

```
[2016-01-06 17:13:10,980][INFO ][repositories             ] [Lazarus] put repository [elasticsearch-snapshots]
[2016-01-06 17:17:39,558][INFO ][snapshots                ] [Lazarus] snapshot [elasticsearch-snapshots:before-migration-to-es-service] is done
```
We've seen speeds of around 1GB-per-min for backup to S3 within the same region.


## Restore Mode

### Restore Data from an Existing S3 Snapshot

The restore operation can be performed on any node of the destination cluster. Restoration can take a while because Elasticsearch will replicate the indices across the destination cluster.

As per the [Elasticsearch Restore Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html#_restore):

> The restore operation can be performed on a functioning cluster. However, an existing index can be only restored if it’s closed and has the same number of shards as the index in the snapshot. The restore operation automatically opens restored indices if they were closed and creates new indices if they didn’t exist in the cluster.

First ensure that you have entered valid values in the `es-s3-snapshot.conf` file. This is critical. See the previous section for details.

Once you have a correct `es-s3-snapshot.conf` file, simply run the utility in `restore` mode on a destination ES node like this:

```sh
 python es-s3-snapshot.py -m restore
```

## Command-Line Help
Use `--help` or `-h` command line parameters to get a summary of the utility with basic usage guidelines.

```sh
$ python es-s3-snapshot.py --help

usage: es-s3-snapshot.py [-h] -m {backup,restore}

Push specified Elasticsearch indices from SOURCE to DESTINATION as per config
in the 'es-s3-snapshot.conf' file.

optional arguments:
  -h, --help            show this help message and exit

required named arguments:
  -m {backup,restore}, --mode {backup,restore}
                        Mode of operation. Choose 'backup' on your SOURCE
                        cluster. Choose 'restore' on your DESTINATION cluster.
```

## Feedback
Comments, suggestions, brickbats: ambarATcloudcover.in

