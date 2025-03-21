# vmbackup

`vmbackup` creates VictoriaMetrics data backups from [instant snapshots](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#how-to-work-with-snapshots).

Supported storage systems for backups:

* [GCS](https://cloud.google.com/storage/). Example: `gs://<bucket>/<path/to/backup>`
* [S3](https://aws.amazon.com/s3/). Example: `s3://<bucket>/<path/to/backup>`
* Any S3-compatible storage such as [MinIO](https://github.com/minio/minio), [Ceph](https://docs.ceph.com/en/pacific/radosgw/s3/) or [Swift](https://platform.swiftstack.com/docs/admin/middleware/s3_middleware.html). See [these docs](#advanced-usage) for details.
* Local filesystem. Example: `fs://</absolute/path/to/backup>`. Note that `vmbackup` prevents from storing the backup into the directory pointed by `-storageDataPath` command-line flag, since this directory should be managed solely by VictoriaMetrics or `vmstorage`.

`vmbackup` supports incremental and full backups. Incremental backups are created automatically if the destination path already contains data from the previous backup.
Full backups can be sped up with `-origin` pointing to an already existing backup on the same remote storage. In this case `vmbackup` makes server-side copy for the shared
data between the existing backup and new backup. It saves time and costs on data transfer.

Backup process can be interrupted at any time. It is automatically resumed from the interruption point when restarting `vmbackup` with the same args.

Backed up data can be restored with [vmrestore](https://docs.victoriametrics.com/vmrestore.html).

See [this article](https://medium.com/@valyala/speeding-up-backups-for-big-time-series-databases-533c1a927883) for more details.

See also [vmbackupmanager](https://docs.victoriametrics.com/vmbackupmanager.html) tool built on top of `vmbackup`. This tool simplifies
creation of hourly, daily, weekly and monthly backups.

## Use cases

### Regular backups

Regular backup can be performed with the following command:

```console
vmbackup -storageDataPath=</path/to/victoria-metrics-data> -snapshot.createURL=http://localhost:8428/snapshot/create -dst=gs://<bucket>/<path/to/new/backup>
```

* `</path/to/victoria-metrics-data>` - path to VictoriaMetrics data pointed by `-storageDataPath` command-line flag in single-node VictoriaMetrics or in cluster `vmstorage`.
  There is no need to stop VictoriaMetrics for creating backups since they are performed from immutable [instant snapshots](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#how-to-work-with-snapshots).
* `http://victoriametrics:8428/snapshot/create` is the url for creating snapshots according to [these docs](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#how-to-work-with-snapshots). `vmbackup` creates a snapshot by querying the provided `-snapshot.createURL`, then performs the backup and then automatically removes the created snapshot.
* `<bucket>` is an already existing name for [GCS bucket](https://cloud.google.com/storage/docs/creating-buckets).
* `<path/to/new/backup>` is the destination path where new backup will be placed.

### Regular backups with server-side copy from existing backup

If the destination GCS bucket already contains the previous backup at `-origin` path, then new backup can be sped up
with the following command:

```console
./vmbackup -storageDataPath=</path/to/victoria-metrics-data> -snapshot.createURL=http://localhost:8428/snapshot/create -dst=gs://<bucket>/<path/to/new/backup> -origin=gs://<bucket>/<path/to/existing/backup>
```

It saves time and network bandwidth costs by performing server-side copy for the shared data from the `-origin` to `-dst`.

### Incremental backups

Incremental backups are performed if `-dst` points to an already existing backup. In this case only new data is uploaded to remote storage.
It saves time and network bandwidth costs when working with big backups:

```console
./vmbackup -storageDataPath=</path/to/victoria-metrics-data> -snapshot.createURL=http://localhost:8428/snapshot/create -dst=gs://<bucket>/<path/to/existing/backup>
```

### Smart backups

Smart backups mean storing full daily backups into `YYYYMMDD` folders and creating incremental hourly backup into `latest` folder:

* Run the following command every hour:

```console
./vmbackup -storageDataPath=</path/to/victoria-metrics-data> -snapshot.createURL=http://localhost:8428/snapshot/create -dst=gs://<bucket>/latest
```

Where `<latest-snapshot>` is the latest [snapshot](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#how-to-work-with-snapshots).
The command will upload only changed data to `gs://<bucket>/latest`.

* Run the following command once a day:

```console
vmbackup -storageDataPath=</path/to/victoria-metrics-data> -snapshot.createURL=http://localhost:8428/snapshot/create -dst=gs://<bucket>/<YYYYMMDD> -origin=gs://<bucket>/latest
```

Where `<daily-snapshot>` is the snapshot for the last day `<YYYYMMDD>`.

This apporach saves network bandwidth costs on hourly backups (since they are incremental) and allows recovering data from either the last hour (`latest` backup)
or from any day (`YYYYMMDD` backups). Note that hourly backup shouldn't run when creating daily backup.

Do not forget to remove old backups when they are no longer needed in order to save storage costs.

See also [vmbackupmanager tool](https://docs.victoriametrics.com/vmbackupmanager.html) for automating smart backups.

## How does it work?

The backup algorithm is the following:

1. Create a snapshot by querying the provided `-snapshot.createURL`
2. Collect information about files in the created snapshot, in the `-dst` and in the `-origin`.
3. Determine which files in `-dst` are missing in the created snapshot, and delete them. These are usually small files, which are already merged into bigger files in the snapshot.
4. Determine which files in the created snapshot are missing in `-dst`. These are usually small new files and bigger merged files.
5. Determine which files from step 3 exist in the `-origin`, and perform server-side copy of these files from `-origin` to `-dst`.
   These are usually the biggest and the oldest files, which are shared between backups.
6. Upload the remaining files from step 3 from the created snapshot to `-dst`.
7. Delete the created snapshot.

The algorithm splits source files into 1 GiB chunks in the backup. Each chunk is stored as a separate file in the backup.
Such splitting balances between the number of files in the backup and the amounts of data that needs to be re-transfered after temporary errors.

`vmbackup` relies on [instant snapshot](https://medium.com/@valyala/how-victoriametrics-makes-instant-snapshots-for-multi-terabyte-time-series-data-e1f3fb0e0282) properties:

* All the files in the snapshot are immutable.
* Old files are periodically merged into new files.
* Smaller files have higher probability to be merged.
* Consecutive snapshots share many identical files.

These properties allow performing fast and cheap incremental backups and server-side copying from `-origin` paths.
See [this article](https://medium.com/@valyala/speeding-up-backups-for-big-time-series-databases-533c1a927883) for more details.
`vmbackup` can work improperly or slowly when these properties are violated.

## Troubleshooting

* If the backup is slow, then try setting higher value for `-concurrency` flag. This will increase the number of concurrent workers that upload data to backup storage.
* If `vmbackup` eats all the network bandwidth, then set `-maxBytesPerSecond` to the desired value.
* If `vmbackup` has been interrupted due to temporary error, then just restart it with the same args. It will resume the backup process.
* Backups created from [single-node VictoriaMetrics](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html) cannot be restored
  at [cluster VictoriaMetrics](https://docs.victoriametrics.com/Cluster-VictoriaMetrics.html) and vice versa.

## Advanced usage

* Obtaining credentials from a file.

  Add flag `-credsFilePath=/etc/credentials` with the following content:

    for s3 (aws, minio or other s3 compatible storages):

     ```console
     [default]
     aws_access_key_id=theaccesskey
     aws_secret_access_key=thesecretaccesskeyvalue
    ```

    for gce cloud storage:

    ```json
    {
           "type": "service_account",
           "project_id": "project-id",
           "private_key_id": "key-id",
           "private_key": "-----BEGIN PRIVATE KEY-----\nprivate-key\n-----END PRIVATE KEY-----\n",
           "client_email": "service-account-email",
           "client_id": "client-id",
           "auth_uri": "https://accounts.google.com/o/oauth2/auth",
           "token_uri": "https://accounts.google.com/o/oauth2/token",
           "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
           "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/service-account-email"
    }
    ```

* Usage with s3 custom url endpoint. It is possible to use `vmbackup` with s3 compatible storages like minio, cloudian, etc.
  You have to add a custom url endpoint via flag:

```console
  # for minio
  -customS3Endpoint=http://localhost:9000

  # for aws gov region
  -customS3Endpoint=https://s3-fips.us-gov-west-1.amazonaws.com
```

* Run `vmbackup -help` in order to see all the available options:

```console
  -concurrency int
     The number of concurrent workers. Higher concurrency may reduce backup duration (default 10)
  -configFilePath string
     Path to file with S3 configs. Configs are loaded from default location if not set.
     See https://docs.aws.amazon.com/general/latest/gr/aws-security-credentials.html
  -configProfile string
     Profile name for S3 configs. If no set, the value of the environment variable will be loaded (AWS_PROFILE or AWS_DEFAULT_PROFILE), or if both not set, DefaultSharedConfigProfile is used
  -credsFilePath string
     Path to file with GCS or S3 credentials. Credentials are loaded from default locations if not set.
     See https://cloud.google.com/iam/docs/creating-managing-service-account-keys and https://docs.aws.amazon.com/general/latest/gr/aws-security-credentials.html
  -customS3Endpoint string
     Custom S3 endpoint for use with S3-compatible storages (e.g. MinIO). S3 is used if not set
  -dst string
     Where to put the backup on the remote storage. Example: gs://bucket/path/to/backup/dir, s3://bucket/path/to/backup/dir or fs:///path/to/local/backup/dir
     -dst can point to the previous backup. In this case incremental backup is performed, i.e. only changed data is uploaded
  -enableTCP6
     Whether to enable IPv6 for listening and dialing. By default only IPv4 TCP and UDP is used
  -envflag.enable
     Whether to enable reading flags from environment variables additionally to command line. Command line flag values have priority over values from environment vars. Flags are read only from command line if this flag isn't set. See https://docs.victoriametrics.com/#environment-variables for more details
  -envflag.prefix string
     Prefix for environment variables if -envflag.enable is set
  -eula
     By specifying this flag, you confirm that you have an enterprise license and accept the EULA https://victoriametrics.com/assets/VM_EULA.pdf
  -flagsAuthKey string
     Auth key for /flags endpoint. It must be passed via authKey query arg. It overrides httpAuth.* settings
  -fs.disableMmap
     Whether to use pread() instead of mmap() for reading data files. By default mmap() is used for 64-bit arches and pread() is used for 32-bit arches, since they cannot read data files bigger than 2^32 bytes in memory. mmap() is usually faster for reading small data chunks than pread()
  -http.connTimeout duration
     Incoming http connections are closed after the configured timeout. This may help to spread the incoming load among a cluster of services behind a load balancer. Please note that the real timeout may be bigger by up to 10% as a protection against the thundering herd problem (default 2m0s)
  -http.disableResponseCompression
     Disable compression of HTTP responses to save CPU resources. By default compression is enabled to save network bandwidth
  -http.idleConnTimeout duration
     Timeout for incoming idle http connections (default 1m0s)
  -http.maxGracefulShutdownDuration duration
     The maximum duration for a graceful shutdown of the HTTP server. A highly loaded server may require increased value for a graceful shutdown (default 7s)
  -http.pathPrefix string
     An optional prefix to add to all the paths handled by http server. For example, if '-http.pathPrefix=/foo/bar' is set, then all the http requests will be handled on '/foo/bar/*' paths. This may be useful for proxied requests. See https://www.robustperception.io/using-external-urls-and-proxies-with-prometheus
  -http.shutdownDelay duration
     Optional delay before http server shutdown. During this delay, the server returns non-OK responses from /health page, so load balancers can route new requests to other servers
  -httpAuth.password string
     Password for HTTP Basic Auth. The authentication is disabled if -httpAuth.username is empty
  -httpAuth.username string
     Username for HTTP Basic Auth. The authentication is disabled if empty. See also -httpAuth.password
  -httpListenAddr string
     TCP address for exporting metrics at /metrics page (default ":8420")
  -loggerDisableTimestamps
     Whether to disable writing timestamps in logs
  -loggerErrorsPerSecondLimit int
     Per-second limit on the number of ERROR messages. If more than the given number of errors are emitted per second, the remaining errors are suppressed. Zero values disable the rate limit
  -loggerFormat string
     Format for logs. Possible values: default, json (default "default")
  -loggerLevel string
     Minimum level of errors to log. Possible values: INFO, WARN, ERROR, FATAL, PANIC (default "INFO")
  -loggerOutput string
     Output for the logs. Supported values: stderr, stdout (default "stderr")
  -loggerTimezone string
     Timezone to use for timestamps in logs. Timezone must be a valid IANA Time Zone. For example: America/New_York, Europe/Berlin, Etc/GMT+3 or Local (default "UTC")
  -loggerWarnsPerSecondLimit int
     Per-second limit on the number of WARN messages. If more than the given number of warns are emitted per second, then the remaining warns are suppressed. Zero values disable the rate limit
  -maxBytesPerSecond size
     The maximum upload speed. There is no limit if it is set to 0
     Supports the following optional suffixes for size values: KB, MB, GB, KiB, MiB, GiB (default 0)
  -memory.allowedBytes size
     Allowed size of system memory VictoriaMetrics caches may occupy. This option overrides -memory.allowedPercent if set to a non-zero value. Too low a value may increase the cache miss rate usually resulting in higher CPU and disk IO usage. Too high a value may evict too much data from OS page cache resulting in higher disk IO usage
     Supports the following optional suffixes for size values: KB, MB, GB, KiB, MiB, GiB (default 0)
  -memory.allowedPercent float
     Allowed percent of system memory VictoriaMetrics caches may occupy. See also -memory.allowedBytes. Too low a value may increase cache miss rate usually resulting in higher CPU and disk IO usage. Too high a value may evict too much data from OS page cache which will result in higher disk IO usage (default 60)
  -metricsAuthKey string
     Auth key for /metrics endpoint. It must be passed via authKey query arg. It overrides httpAuth.* settings
  -origin string
     Optional origin directory on the remote storage with old backup for server-side copying when performing full backup. This speeds up full backups
  -pprofAuthKey string
     Auth key for /debug/pprof/* endpoints. It must be passed via authKey query arg. It overrides httpAuth.* settings
  -pushmetrics.extraLabels array
     Optional labels to add to metrics pushed to -pushmetrics.url . For example, -pushmetrics.extraLabels='instance="foo"' adds instance="foo" label to all the metrics pushed to -pushmetrics.url
     Supports an array of values separated by comma or specified via multiple flags.
  -pushmetrics.interval duration
     Interval for pushing metrics to -pushmetrics.url (default 10s)
  -pushmetrics.url array
     Optional URL to push metrics exposed at /metrics page. See https://docs.victoriametrics.com/#push-metrics . By default metrics exposed at /metrics page aren't pushed to any remote storage
     Supports an array of values separated by comma or specified via multiple flags.
  -s3ForcePathStyle
     Prefixing endpoint with bucket name when set false, true by default. (default true)
  -snapshot.createURL string
     VictoriaMetrics create snapshot url. When this is given a snapshot will automatically be created during backup. Example: http://victoriametrics:8428/snapshot/create . There is no need in setting -snapshotName if -snapshot.createURL is set
  -snapshot.deleteURL string
     VictoriaMetrics delete snapshot url. Optional. Will be generated from -snapshot.createURL if not provided. All created snapshots will be automatically deleted. Example: http://victoriametrics:8428/snapshot/delete
  -snapshotName string
     Name for the snapshot to backup. See https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#how-to-work-with-snapshots. There is no need in setting -snapshotName if -snapshot.createURL is set
  -storageDataPath string
     Path to VictoriaMetrics data. Must match -storageDataPath from VictoriaMetrics or vmstorage (default "victoria-metrics-data")
  -tls
     Whether to enable TLS for incoming HTTP requests at -httpListenAddr (aka https). -tlsCertFile and -tlsKeyFile must be set if -tls is set
  -tlsCertFile string
     Path to file with TLS certificate if -tls is set. Prefer ECDSA certs instead of RSA certs as RSA certs are slower. The provided certificate file is automatically re-read every second, so it can be dynamically updated
  -tlsCipherSuites array
     Optional list of TLS cipher suites for incoming requests over HTTPS if -tls is set. See the list of supported cipher suites at https://pkg.go.dev/crypto/tls#pkg-constants
     Supports an array of values separated by comma or specified via multiple flags.
  -tlsKeyFile string
     Path to file with TLS key if -tls is set. The provided key file is automatically re-read every second, so it can be dynamically updated
  -version
     Show VictoriaMetrics version
```

## How to build from sources

It is recommended using [binary releases](https://github.com/VictoriaMetrics/VictoriaMetrics/releases) - see `vmutils-*` archives there.

### Development build

1. [Install Go](https://golang.org/doc/install). The minimum supported version is Go 1.17.
2. Run `make vmbackup` from the root folder of [the repository](https://github.com/VictoriaMetrics/VictoriaMetrics).
   It builds `vmbackup` binary and puts it into the `bin` folder.

### Production build

1. [Install docker](https://docs.docker.com/install/).
2. Run `make vmbackup-prod` from the root folder of [the repository](https://github.com/VictoriaMetrics/VictoriaMetrics).
   It builds `vmbackup-prod` binary and puts it into the `bin` folder.

### Building docker images

Run `make package-vmbackup`. It builds `victoriametrics/vmbackup:<PKG_TAG>` docker image locally.
`<PKG_TAG>` is auto-generated image tag, which depends on source code in the repository.
The `<PKG_TAG>` may be manually set via `PKG_TAG=foobar make package-vmbackup`.

The base docker image is [alpine](https://hub.docker.com/_/alpine) but it is possible to use any other base image
by setting it via `<ROOT_IMAGE>` environment variable. For example, the following command builds the image on top of [scratch](https://hub.docker.com/_/scratch) image:

```console
ROOT_IMAGE=scratch make package-vmbackup
```
