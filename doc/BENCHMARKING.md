## Benchmarking

Mountpoint for Amazon S3 is a simple, high-throughput file client for mounting an Amazon S3 bucket as a local file system.
To avoid new changes introducing performance regressions, we run a performance benchmark on every commit using [fio](https://github.com/axboe/fio), an awesome open-source application for file system benchmarking.

### Workloads

***read workload*** - we measure two aspects of the read operation, throughput (with and without caching enabled) and latency. For the first part, we use fio to simulate IO workloads for sequential read or random read for a specific duration then measure their throughput. On the latency side, we are using time to first byte as data points by running workloads that read one byte off of existing files on Mountpoint and measure the time it takes to complete the operation. Each of the test is defined in a separate .fio file, and the file name indicates what is the test case for that file, for example `seq_read.fio` is the benchmark for sequential read. All of fio configuration files can be found at path [mountpoint-s3/scripts/fio/read/](../mountpoint-s3/scripts/fio/read) and [mountpoint-s3/scripts/fio/read_latency/](../mountpoint-s3/scripts/fio/read_latency).

In general, we run each IO operation for 30 seconds against a 100 GiB file. But there are some variants in configuration where we also want to test to see how Mountpoint would perform with these configurations. Here is the list of all the variants we have tested.

* **four_threads**: running the workload concurrently by spawning four fio threads to do the same job.
* **direct_io**: bypassing kernel page cache by opening the files with `O_DIRECT` option. This option is only available on Linux.
* **small_file**: run the IO operation against smaller files (5 MiB instead of 100 GiB).

***readdir workload*** - we measure how long it takes to run `ls` command against directories with different size. Each directory has no subdirectory and contains a specific number of files, range from 100 to 100000 files, which we have to create manually using fio then upload them to S3 bucket before running the benchmark. The fio configuration files for creating them can be found at path [mountpoint-s3/scripts/fio/create/](../mountpoint-s3/scripts/fio/create).

***write workload*** - we measure write throughput by using fio to simulate sequential write workloads. The fio configuration files for write workloads can be found at path [mountpoint-s3/scripts/fio/write/](../mountpoint-s3/scripts/fio/write).

### Regression Testing
Our CI runs the benchmark automatically for any new commits to the main branch or specific pull requests that we have reviewed and tagged with **performance** label. Every benchmark from the CI workflow will be running on `m5dn.24xlarge` EC2 instances (100 Gbps network speed) with Ubuntu 22.04 in us-east-1 against a bucket in us-east-1.

We keep the records of benchmarking results in `gh-pages` branch and the performance charts are available for viewing:
- [throughput chart](https://awslabs.github.io/mountpoint-s3/dev/bench/)
- [throughput (with cache) chart](https://awslabs.github.io/mountpoint-s3/dev/cache_bench/)
- [throughput (s3-express) chart](https://awslabs.github.io/mountpoint-s3/dev/s3-express/bench)
- [latency chart](https://awslabs.github.io/mountpoint-s3/dev/latency_bench/)
- [latency (s3-express) chart](https://awslabs.github.io/mountpoint-s3/dev/s3-express/latency_bench/)

### Running the benchmark
While our benchmark script is written for CI testing only, it is possible to run manually.
You can use the following steps.

1. Install dependencies and configure FUSE by running the following script in the `mountpoint-s3` repository:

        bash .github/actions/install-dependencies/install.sh \
                --fuse-version 2 \
                --with-fio --with-libunwind

2. Set environment variables related to the benchmark. There are four required environment variables you need to set in order to run the benchmark.

        export S3_BUCKET_NAME=bucket_name
        export S3_BUCKET_TEST_PREFIX=prefix_path/
        export S3_ENDPOINT_URL=endpoint_url # optional
        # these filenames only needed by cache benchmark
        export S3_BUCKET_BENCH_FILE=bench_file_name
        export S3_BUCKET_SMALL_BENCH_FILE=small_bench_file_name
        # to filter by job name; e.g. to only run small jobs
        export S3_JOB_NAME_FILTER=small
        # when running locally, we skip setting up local EC2 storage by default; to override that you can pass the following (value of the variable does not matter)
        export S3_MOUNT_LOCAL_STORAGE=yes

3. Run the benchmark script for [throughput](../mountpoint-s3/scripts/fs_bench.sh), [throughput with caching](../mountpoint-s3/scripts/fs_cache_bench.sh) or [latency](../mountpoint-s3/scripts/fs_latency_bench.sh).

        # to run throughput benchmarks
        ./mountpoint-s3/scripts/fs_bench.sh
        # to run throughput benchmarks with caching enabled
        ./mountpoint-s3/scripts/fs_cache_bench.sh
        # to run latency benchmarks
        ./mountpoint-s3/scripts/fs_latency_bench.sh

4. You should see the benchmark logs in `bench.out` file in the project root directory. The combined results will be saved into a JSON file at `results/output.json`.
