---
id: cluster-deployment-guide
title: Temporal Cluster deployment guide
sidebar_label: Cluster deployment
description: This section provides an overview of how to deploy and operate a Temporal Cluster in a live environment.
toc_max_heading_level: 4
---

<!-- THIS FILE IS GENERATED. DO NOT EDIT THIS FILE DIRECTLY -->

This section provides an overview of how to deploy and operate a Temporal Cluster in a live environment.

:::info WORK IN PROGRESS

This guide is a work in progress.
Some sections may be incomplete.
Information may change at any time.

Legacy production deployment information is available [here](/server/production-deployment)

:::

## Advanced Visibility

[Advanced Visibility](/visibility#advanced-visibility) features depend on an integration with Elasticsearch.

To integrate Elasticsearch with your Temporal Cluster, edit the `persistence` section of your `development.yaml` configuration file and run the index schema setup commands.

:::note

These steps are needed only if you have a "plain" [Temporal Server Docker image](https://hub.docker.com/r/temporalio/server).

If you operate a Temporal Cluster using our [Helm charts](https://github.com/temporalio/helm-charts) or
[docker-compose](https://github.com/temporalio/docker-compose), the Elasticsearch index schema and index are created automatically using the [auto-setup Docker image](https://hub.docker.com/r/temporalio/auto-setup).

:::

:::note Supported versions

- Elasticsearch v7.10 is supported from Temporal version 1.7.0 onwards
- Elasticsearch v6.8 is supported in all Temporal versions
- Both versions are explicitly supported with AWS Elasticsearch

:::

#### Edit persistence

1. Add the `advancedVisibilityStore: es-visibility` key-value pair to the `persistence` section.
   The [development_es.yaml](https://github.com/temporalio/temporal/blob/master/config/development_es.yaml) file in the `temporalio/temporal` repo is a working example.
   The configuration instructs the Temporal Cluster how and where to connect to Elasticsearch storage.

```yaml
persistence:
  ...
  advancedVisibilityStore: es-visibility
```

2. Define the Elasticsearch datastore connection information under the `es-visibility` key:

```yaml
persistence:
  ...
  advancedVisibilityStore: es-visibility
  datastores:
    ...
    es-visibility:
      elasticsearch:
        version: "v7"
        url:
          scheme: "http"
          host: "127.0.0.1:9200"
        indices:
          visibility: temporal_visibility_v1_dev
```

#### Create index schema and index

Run the following commands to create the index schema and index:

```bash
# ES_SERVER is the URL of Elasticsearch server; for example, "http://localhost:9200".
SETTINGS_URL="${ES_SERVER}/_cluster/settings"
SETTINGS_FILE=${TEMPORAL_HOME}/schema/elasticsearch/visibility/cluster_settings_${ES_VERSION}.json
TEMPLATE_URL="${ES_SERVER}/_template/temporal_visibility_v1_template"
SCHEMA_FILE=${TEMPORAL_HOME}/schema/elasticsearch/visibility/index_template_${ES_VERSION}.json
INDEX_URL="${ES_SERVER}/${ES_VIS_INDEX}"
curl --fail --user "${ES_USER}":"${ES_PWD}" -X PUT "${SETTINGS_URL}" -H "Content-Type: application/json" --data-binary "@${SETTINGS_FILE}" --write-out "\n"
curl --fail --user "${ES_USER}":"${ES_PWD}" -X PUT "${TEMPLATE_URL}" -H 'Content-Type: application/json' --data-binary "@${SCHEMA_FILE}" --write-out "\n"
curl --user "${ES_USER}":"${ES_PWD}" -X PUT "${INDEX_URL}" --write-out "\n"
```

#### Set Elasticsearch privileges

Ensure that the following privileges are granted for the Elasticsearch Temporal index:

- **Read**
  - [index privileges](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-privileges.html#privileges-list-indices): `create`, `index`, `delete`, `read`
- **Write**
  - [index privileges](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-privileges.html#privileges-list-indices): `write`
- **Custom Search Attributes**
  - [index privileges](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-privileges.html#privileges-list-indices): `manage`
  - [cluster privileges](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-privileges.html#privileges-list-cluster): `monitor` or `manage`.

#### Add custom Search Attributes (optional)

This step is optional.
Here we are adding custom Search Attributes to your Cluster.

Run the following `tctl` command to add the `ProductId` custom Search Attribute to the Temporal Cluster (and Elasticsearch Temporal index):

```bash
tctl admin cluster add-search-attributes --name ProductId --type Keyword
```

Run the following `tctl` command to add custom Search Attributes used by samples and SDK integration tests:

```bash
tctl --auto_confirm admin cluster add-search-attributes \
    --name CustomKeywordField --type Keyword \
    --name CustomStringField --type Text \
    --name CustomTextField --type Text \
    --name CustomIntField --type Int \
    --name CustomDatetimeField --type Datetime \
    --name CustomDoubleField --type Double \
    --name CustomBoolField --type Bool
```

## Archival

[Archival](/clusters#archival) is a feature that automatically backs up Workflow Execution Event Histories and Visibility data from Temporal Cluster persistence to a custom blob store.

### Set up

[Archival](/clusters#archival) consists of the following elements:

- **Configuration**: Archival is controlled by the [server configuration](https://github.com/temporalio/temporal/blob/master/config/development.yaml#L81) (i.e. the `config/development.yaml` file).
- **Provider**: Location where the data should be archived. Supported providers are S3, GCloud, and the local file system.
- **URI**: Specifies which provider should be used. The system uses the URI schema and path to make the determination.

Take the following steps to set up Archival:

1. [Set up the provider](#providers) of your choice.
2. [Configure Archival](#configuration).
3. [Create a Namespace](#namespace-creation) that uses a valid URI and has Archival enabled.

#### Providers

Temporal directly supports several providers:

- **Local file system**: The [filestore archiver](https://github.com/temporalio/temporal/tree/master/common/archiver/filestore) is used to archive data in the file system of whatever host the Temporal server is running on. This provider is used mainly for local installations and testing and should not be relied on for production environments.
- **Google Cloud**: The [gcloud archiver](https://github.com/temporalio/temporal/tree/master/common/archiver/gcloud) is used to connect and archive data with [Google Cloud](https://cloud.google.com/storage).
- **S3**: The [s3store archiver](https://github.com/temporalio/temporal/tree/master/common/archiver/s3store) is used to connect and archive data with [S3](https://aws.amazon.com/s3).
- **Custom**: If you want to use a provider that is not currently supported, you can [create your own archiver](#custom-archiver) to support it.

Make sure that you save the provider's storage location URI in a place where you can reference it later, because it is passed as a parameter when you [create a Namespace](#namespace-creation).

#### Configuration

Archival configuration is defined in the [`config/development.yaml`](https://github.com/temporalio/temporal/blob/master/config/development.yaml#L93) file.
Let's look at an example configuration:

```yaml
# Cluster level Archival config
archival:
  # Event History configuration
  history:
    # Archival is enabled at the cluster level
    state: "enabled"
    enableRead: true
    # Namespaces can use either the local filestore provider or the Google Cloud provider
    provider:
      filestore:
        fileMode: "0666"
        dirMode: "0766"
      gstorage:
        credentialsPath: "/tmp/gcloud/keyfile.json"

# Default values for a Namespace if none are provided at creation
namespaceDefaults:
  # Archival defaults
  archival:
    # Event History defaults
    history:
      state: "enabled"
      # New Namespaces will default to the local provider
      URI: "file:///tmp/temporal_archival/development"
```

You can disable Archival by setting `archival.history.state` and `namespaceDefaults.archival.history.state` to `"disabled"`.

Example:

```yaml
archival:
  history:
    state: "disabled"

namespaceDefaults:
  archival:
    history:
      state: "disabled"
```

The following table showcases acceptable values for each configuration and what purpose they serve.

| Config                                         | Acceptable values                                                                  | Description                                                                                                                  |
| ---------------------------------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `archival.history.state`                       | `enabled`, `disabled`                                                              | Must be `enabled` to use the Archival feature with any Namespace in the cluster.                                             |
| `archival.history.enableRead`                  | `true`, `false`                                                                    | Must be `true` to read from the archived Event History.                                                                      |
| `archival.history.provider`                    | Sub provider configs are `filestore`, `gstorage`, `s3`, or `your_custom_provider`. | Default config specifies `filestore`.                                                                                        |
| `archival.history.provider.filestore.fileMode` | File permission string                                                             | File permissions of the archived files. We recommend using the default value of `"0666"` to avoid read/write issues.         |
| `archival.history.provider.filestore.dirMode`  | File permission string                                                             | Directory permissions of the archive directory. We recommend using the default value of `"0766"` to avoid read/write issues. |
| `namespaceDefaults.archival.history.state`     | `enabled`, `disabled`                                                              | Default state of the Archival feature whenever a new Namespace is created without specifying the Archival state.             |
| `namespaceDefaults.archival.history.URI`       | Valid URI                                                                          | Must be a URI of the file store location and match a schema that correlates to a provider.                                   |

#### Namespace creation

Although Archival is configured at the cluster level, it operates independently within each Namespace.
If an Archival URI is not specified when a Namespace is created, the Namespace uses the value of `defaultNamespace.archival.history.URI` from the `config/development.yaml` file.
The Archival URI cannot be changed after the Namespace is created.
Each Namespace supports only a single Archival URI, but each Namespace can use a different URI.
A Namespace can safely switch Archival between `enabled` and `disabled` states as long as Archival is enabled at the cluster level.

Archival is supported in [Global Namespaces](/concepts/what-is-a-global-namespace/) (Namespaces that span multiple clusters).
When Archival is running in a Global Namespace, it first runs on the active cluster; later it runs on the standby cluster. Before archiving, a history check is done to see what has been previously archived.

#### Test setup

To test Archival locally, start by running a Temporal server:

```bash
./temporal-server start
```

Then register a new Namespace with Archival enabled.

```bash
./tctl --ns samples-namespace namespace register --gd false --history_archival_state enabled --retention 3
```

:::note

If the retention period isn't set, it defaults to 2 days.
The minimum retention period is 1 day.
The maximum retention period is 30 days.

Setting the retention period to 0 results in the error _A valid retention period is not set on request_.

:::

Next, run a sample Workflow such as the [helloworld temporal sample](https://github.com/temporalio/temporal-go-samples/tree/master/helloworld).

When execution is finished, Archival occurs.

#### Retrieve archives

You can retrieve archived Event Histories by copying the `workflowId` and `runId` of the completed Workflow from the log output and running the following command:

```bash
./temporal --ns samples-namespace wf show --wid <workflowId> --rid <runId>
```

### Custom Archiver

To archive data with a given provider, using the [Archival](/clusters#archival) feature, Temporal must have a corresponding Archiver component installed.
The platform does not limit you to the existing providers.
To use a provider that is not currently supported, you can create your own Archiver.

#### Create a new package

The first step is to create a new package for your implementation in [/common/archiver](https://github.com/temporalio/temporal/tree/master/common/archiver).
Create a directory in the archiver folder and arrange the structure to look like the following:

```
temporal/common/archiver
  - filestore/                      -- Filestore implementation
  - provider/
      - provider.go                 -- Provider of archiver instances
  - yourImplementation/
      - historyArchiver.go          -- HistoryArchiver implementation
      - historyArchiver_test.go     -- Unit tests for HistoryArchiver
      - visibilityArchiver.go       -- VisibilityArchiver implementations
      - visibilityArchiver_test.go  -- Unit tests for VisibilityArchiver
```

#### Archiver interfaces

Next, define objects that implement the [HistoryArchiver](https://github.com/temporalio/temporal/blob/master/common/archiver/interface.go#L80) and the [VisibilityArchiver](https://github.com/temporalio/temporal/blob/master/common/archiver/interface.go#L121) interfaces.

The objects should live in `historyArchiver.go` and `visibilityArchiver.go`, respectively.

#### Update provider

Update the `GetHistoryArchiver` and `GetVisibilityArchiver` methods of the `archiverProvider` object in the [/common/archiver/provider/provider.go](https://github.com/temporalio/temporal/blob/master/common/archiver/provider/provider.go) file so that it knows how to create an instance of your archiver.

#### Add configs

Add configs for your archiver to the `config/development.yaml` file and then modify the [HistoryArchiverProvider](https://github.com/temporalio/temporal/blob/master/common/config/config.go#L376) and [VisibilityArchiverProvider](https://github.com/temporalio/temporal/blob/master/common/config/config.go#L393) structs in `/common/common/config.go` accordingly.

#### Custom archiver FAQ

**If my custom Archive method can automatically be retried by the caller, how can I record and access progress between retries?**

Handle this situation by using `ArchiverOptions`.
Here is an example:

```go
func(a * Archiver) Archive(ctx context.Context, URI string, request * ArchiveRequest, opts...ArchiveOption) error {
    featureCatalog: = GetFeatureCatalog(opts...) // this function is defined in options.go
    var progress progress
    // Check if the feature for recording progress is enabled.
    if featureCatalog.ProgressManager != nil {
        if err: = featureCatalog.ProgressManager.LoadProgress(ctx, & prevProgress);
        err != nil {
            // log some error message and return error if needed.
        }
    }

    // Your archiver implementation...

    // Record current progress
    if featureCatalog.ProgressManager != nil {
        if err: = featureCatalog.ProgressManager.RecordProgress(ctx, progress);
        err != nil {
            // log some error message and return error if needed.
        }
    }
}
```

**If my `Archive` method encounters an error that is non-retryable, how do I indicate to the caller that it should not retry?**

```go
func(a * Archiver) Archive(ctx context.Context, URI string, request * ArchiveRequest, opts...ArchiveOption) error {
    featureCatalog: = GetFeatureCatalog(opts...) // this function is defined in options.go

    err: = youArchiverImpl()

    if nonRetryableErr(err) {
        if featureCatalog.NonRetryableError != nil {
            return featureCatalog.NonRetryableError() // when the caller gets this error type back it will not retry anymore.
        }
    }
}
```

**How does my history archiver implementation read history?**

The archiver package provides a utility called [HistoryIterator](https://github.com/temporalio/temporal/blob/master/common/archiver/historyIterator.go) which is a wrapper of [ExecutionManager](https://github.com/temporalio/temporal/blob/master/common/persistence/dataInterfaces.go#L1014).
`HistoryIterator` is more simple than the `HistoryManager`, which is available in the BootstrapContainer, so archiver implementations can choose to use it when reading Workflow histories.
See the [historyIterator.go](https://github.com/temporalio/temporal/blob/master/common/archiver/historyIterator.go) file for more details.
Use the [filestore historyArchiver implementation](https://github.com/temporalio/temporal/tree/master/common/archiver/filestore) as an example.

**Should my archiver define its own error types?**

Each archiver is free to define and return its own errors.
However, many common errors that exist between archivers are already defined in [common/archiver/constants.go](https://github.com/temporalio/temporal/blob/master/common/archiver/constants.go).

**Is there a generic query syntax for the visibility archiver?**

Currently, no.
But this is something we plan to do in the future.
As for now, try to make your syntax similar to the one used by our advanced list Workflow API.

- [s3store](https://github.com/temporalio/temporal/tree/master/common/archiver/s3store#visibility-query-syntax)
- [gcloud](https://github.com/temporalio/temporal/tree/master/common/archiver/gcloud#visibility-query-syntax)

## Upgrade Server version

If a newer version of the [Temporal Server](/clusters#temporal-server) is available, a notification appears in the Temporal Web UI.

:::info

If you are using a version that is older than 1.0.0, reach out to us at [community.temporal.io](http://community.temporal.io) to ask how to upgrade.

:::

First check to see if an upgrade to the database schema is required for the version you wish to upgrade to.
If a database schema upgrade is required, it will be called out directly in the [release notes](https://github.com/temporalio/temporal/releases).
Some releases require changes to the schema, and some do not.
We ensure that any consecutive versions are compatible in terms of database schema upgrades, features, and system behavior, however there is no guarantee that there is compatibility between _any_ 2 non-consecutive versions.

Use one of the upgrade tools to upgrade your database schema to be compatible with the Temporal Server version being upgraded to.

If you are using a schema tools version prior to 1.8.0, we strongly recommend _never_ using the "dryrun" (`-y`, or `--dryrun`) options in any of your schema update commands.
Using this option might lead to potential loss of data, as when using it will create a new database and drop your
existing one.
This flag was removed in the 1.8.0 release.

### Upgrade Cassandra schema

If you are using Cassandra for your Cluster's persistence, use the `temporal-cassandra-tool` to upgrade both the default and visibility schemas.

**Example default schema upgrade:**

```bash
temporal_v1.2.1 $ temporal-cassandra-tool \
   --tls \
   --tls-ca-file <...> \
   --user <cassandra-user> \
   --password <cassandra-password> \
   --endpoint <cassandra.example.com> \
   --keyspace temporal \
   --timeout 120 \
   update \
   --schema-dir ./schema/cassandra/temporal/versioned

```

**Example visibility schema upgrade:**

```bash
temporal_v1.2.1 $ temporal-cassandra-tool \
   --tls \
   --tls-ca-file <...> \
   --user <cassandra-user> \
   --password <cassandra-password> \
   --endpoint <cassandra.example.com> \
   --keyspace temporal_visibility \
   --timeout 120 \
   update \
   --schema-dir ./schema/cassandra/visibility/versioned

```

### Upgrade MySQL / PostgreSQL schema

If you are using MySQL or PostgreSQL use the `temporal-sql-tool`, which works similarly to the `temporal-cassandra-tool`.

Refer to this [Makefile](https://github.com/temporalio/temporal/blob/v1.4.1/Makefile#L367-L383) for context.

#### PostgreSQL

**Example default schema upgrade:**

```bash
./temporal-sql-tool \
	--tls \
	--tls-enable-host-verification \
	--tls-cert-file <path to your client cert> \
	--tls-key-file <path to your client key> \
	--tls-ca-file <path to your CA> \
	--ep localhost -p 5432 -u temporal -pw temporal --pl postgres --db temporal update-schema -d ./schema/postgresql/v96/temporal/versioned
```

**Example visibility schema upgrade:**

```bash
./temporal-sql-tool \
	--tls \
	--tls-enable-host-verification \
	--tls-cert-file <path to your client cert> \
	--tls-key-file <path to your client key> \
	--tls-ca-file <path to your CA> \
	--ep localhost -p 5432 -u temporal -pw temporal --pl postgres --db temporal_visibility update-schema -d ./schema/postgresql/v96/visibility/versioned
```

#### MySQL

**Example default schema upgrade:**

```bash
./temporal-sql-tool \
	--tls \
	--tls-enable-host-verification \
	--tls-cert-file <path to your client cert> \
	--tls-key-file <path to your client key> \
	--tls-ca-file <path to your CA> \
	--ep localhost -p 3036 -u root -pw root --pl mysql --db temporal update-schema -d ./schema/mysql/v57/temporal/versioned/
```

**Example visibility schema upgrade:**

```bash
./temporal-sql-tool \
	--tls \
	--tls-enable-host-verification \
	--tls-cert-file <path to your client cert> \
	--tls-key-file <path to your client key> \
	--tls-ca-file <path to your CA> \
	--ep localhost -p 3036 -u root -pw root --pl mysql --db temporal_visibility update-schema -d ./schema/mysql/v57/visibility/versioned/
```

### Roll-out technique

We recommend preparing a staging Cluster and then do the following to verify the upgrade is successful:

1. Create some simulation load on the staging cluster.
2. Upgrade the database schema in the staging cluster.
3. Wait and observe for a few minutes to verify that there is no unstable behavior from both the server and the simulation load logic.
4. Upgrade the server.
5. Now do the same to the live environment cluster.

## Multi-Cluster Replication

The [Multi-Cluster Replication](/clusters#multi-cluster-replication) feature asynchronously replicates Workflow Execution Event Histories from active Clusters to other passive Clusters, and can be enabled by setting the appropriate values in the `clusterMetadata` section of your configuration file.

1. `enableGlobalNamespace` must be set to `true`.
2. `failoverVersionIncrement` has to be equal across connected Clusters.
3. `initialFailoverVersion` in each Cluster has to assign a different value.
   No equal value is allowed across connected Clusters.

After the above conditions are satisfied, you can start to configure a multi-cluster setup.

#### Set up Multi-Cluster Replication prior to v1.14

You can set this up with [`clusterMetadata` configuration](/references/configuration#clustermetadata); however, this is meant to be only a conceptual guide rather than a detailed tutorial.
Please reach out to us if you need to set this up.

For example:

```yaml
# cluster A
clusterMetadata:
  enableGlobalNamespace: false
  failoverVersionIncrement: 100
  masterClusterName: "clusterA"
  currentClusterName: "clusterA"
  clusterInformation:
    clusterA:
      enabled: true
      initialFailoverVersion: 1
      rpcAddress: "127.0.0.1:7233"
    clusterB:
      enabled: true
      initialFailoverVersion: 2
      rpcAddress: "127.0.0.1:8233"

# cluster B
clusterMetadata:
  enableGlobalNamespace: false
  failoverVersionIncrement: 100
  masterClusterName: "clusterA"
  currentClusterName: "clusterB"
  clusterInformation:
    clusterA:
      enabled: true
      initialFailoverVersion: 1
      rpcAddress: "127.0.0.1:7233"
    clusterB:
      enabled: true
      initialFailoverVersion: 2
      rpcAddress: "127.0.0.1:8233"
```

#### Set up Multi-Cluster Replication in v1.14 and later

You still need to set up local cluster [`clusterMetadata` configuration](/references/configuration#clustermetadata)

For example:

```yaml
# cluster A
clusterMetadata:
  enableGlobalNamespace: false
  failoverVersionIncrement: 100
  masterClusterName: "clusterA"
  currentClusterName: "clusterA"
  clusterInformation:
    clusterA:
      enabled: true
      initialFailoverVersion: 1
      rpcAddress: "127.0.0.1:7233"

# cluster B
clusterMetadata:
  enableGlobalNamespace: false
  failoverVersionIncrement: 100
  masterClusterName: "clusterB"
  currentClusterName: "clusterB"
  clusterInformation:
    clusterB:
      enabled: true
      initialFailoverVersion: 2
      rpcAddress: "127.0.0.1:8233"
```

Then you can use the `tctl admin` tool to add cluster connections. All operations should be executed in both Clusters.

```shell
# Add cluster B connection into cluster A
tctl -address 127.0.0.1:7233 admin cluster upsert-remote-cluster --frontend_address "localhost:8233"
# Add cluster A connection into cluster B
tctl -address 127.0.0.1:8233 admin cluster upsert-remote-cluster --frontend_address "localhost:7233"

# Disable connections
tctl -address 127.0.0.1:7233 admin cluster upsert-remote-cluster --frontend_address "localhost:8233" --enable_connection false
tctl -address 127.0.0.1:8233 admin cluster upsert-remote-cluster --frontend_address "localhost:7233" --enable_connection false

# Delete connections
tctl -address 127.0.0.1:7233 admin cluster remove-remote-cluster --cluster "clusterB"
tctl -address 127.0.0.1:8233 admin cluster remove-remote-cluster --cluster "clusterA"
```
