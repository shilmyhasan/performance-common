# Common Artifacts for Performance Tests

---
|  Branch | Build Status |
| :------ |:------------ |
| master  | [![Build Status](https://wso2.org/jenkins/buildStatus/icon?job=platform-builds/performance-common)](https://wso2.org/jenkins/job/platform-builds/job/performance-common/) |
---

This repository has common artifacts to be used for Performance Tests.

In [components](components), there are several Java projects and each project builds an executable JAR file.

The [distribution](distribution) directory has the scripts and the Maven project to build the final distribution package
 including all scripts and components to be used for performance tests.

The package (**performance-common-distribution-${version}.tar.gz**) built by the distribution maven module is the
 only package required for performance tests from this repository.

This package only provides helper scripts and applications. You should extend the functionality of these scripts to run
performance tests.

It's recommended to include the contents of this package with any scripts written to extend the functionality.

## Package contents

Following is the tree view of the contents inside distribution package.

```
|-- java
|   `-- install-java.sh
|-- jmeter
|   |-- create-summary-csv.sh
|   |-- csv-to-markdown-converter.py
|   |-- install-jmeter.sh
|   |-- jmeter-server-start.sh
|   |-- perf-test-common.sh
|   `-- user.properties
|-- jtl-splitter
|   |-- jtl-splitter-${version}.jar
|   `-- jtl-splitter.sh
|-- netty-service
|   |-- netty-http-echo-service-${version}.jar
|   `-- netty-start.sh
|-- payloads
|   |-- generate-payloads.sh
|   `-- payload-generator-${version}.jar
|-- sar
|   `-- install-sar.sh
`-- setup
    |-- setup-common.sh
    |-- setup-jmeter-client.sh
    |-- setup-jmeter.sh
    `-- setup-netty.sh

```

Each directory has one or more executable scripts. All scripts support `-h` (help) option.

**Note:** Most of these scripts will work only on Debian based systems like Ubuntu.

See following sections for more details.

### Java

Use the `install-java.sh` script to install Oracle Java Development Kit (JDK) on 64bit Linux.

The `install-java.sh` script in this directory will not be useful when OpenJDK is used. It's recommended to use the default package
repositories to install OpenJDK.

Currently `install-java.sh` script supports installing Oracle JDK 8.

You must download latest [JDK 8](http://www.oracle.com/technetwork/java/javase/downloads/index.html).

This script can also install Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy files. You need to
 copy JCE Policy zip file to the same location as the downloaded JDK file (tar.gz)

The script needs to be run as root. The JDK will be extracted to `/usr/lib/jvm` directory.

```console
ubuntu@server:~$ ./java/install-java.sh -h

Usage: 
./java/install-java.sh -f <java_dist> [-p <java_dir>] [-h]

-f: The jdk tar.gz file.
-p: Java installation directory.
-h: Display this help and exit.
```

### JMeter

#### Installing JMeter

The `install-jmeter.sh` script will extract JMeter, install Plugin Manager and copy the `user.properties` file.

The `user.properties` file has recommended configurations for performance tests.

There is an option to download latest [Apache JMeter](http://jmeter.apache.org/download_jmeter.cgi).

```console
ubuntu@server:~$ ./jmeter/install-jmeter.sh -h

Usage: 
./jmeter/install-jmeter.sh -i <installation_dir> [-f <jmeter_dist>] [-d] [-p <jmeter_plugin_name>] [-h]

-i: The JMeter installation directory.
-f: The JMeter tgz distribution.
-d: Download JMeter from web.
-p: The name of the JMeter Plugin to install. You can provide multiple names.
-h: Display this help and exit.
```

#### Running standard server performance tests.

You can extend `perf-test-common.sh` to run standard server performance tests. It supports testing with multiple concurrent
users, different message sizes, different backend service delays and different heap memory sizes of the target server.

The script also supports running remote (distributed) JMeter tests and it will also summarize the results for warmup and
measurement periods.

A script can extend this `perf-test-common.sh` script by simply sourcing the `perf-test-common.sh` script.

For example:

```bash
# Execute common script
. $script_dir/perf-test-common.sh
```

Any script depending on this script must define test scenarios as follows:

```bash
declare -A test_scenario0=(
    [name]="test_scenario_name1"
    [jmx]="test_scenario_name1.jmx"
    [use_backend]=true
    [skip]=false
)
declare -A test_scenario1=(
    [name]="test_scenario_name2"
    [jmx]="test_scenario_name2.jmx"
    [use_backend]=true
    [skip]=false
)
```

Then define following functions in the script.
1. `before_execute_test_scenario`
2. `after_execute_test_scenario`

In above functions, following variables may be used
1. `scenario_name`
2. `heap`
3. `users`
4. `msize`
5. `sleep_time`
6. `report_location`

Use `jmeter_params` array in `before_execute_test_scenario` to provide JMeter parameters.

In before_execute_test_scenario `JMETER_JVM_ARGS` variable can be set to provide
additional JVM arguments to JMeter.

Finally, execute test scenarios using the function `test_scenarios`.


```console
ubuntu@server:~$ ./jmeter/perf-test-common.sh -h

Usage: 
./jmeter/perf-test-common.sh [-u <concurrent_users>] [-b <message_sizes>] [-s <sleep_times>] [-m <heap_sizes>] [-d <test_duration>] [-w <warmup_time>]
   [-n <jmeter_servers>] [-j <jmeter_server_heap_size>] [-k <jmeter_client_heap_size>] [-l <netty_service_heap_size>]
   [-i <include_scenario_name>] [-e <include_scenario_name>] [-t] [-p <estimated_processing_time_in_between_tests>] [-h]

-u: Concurrent Users to test. You can give multiple options to specify multiple users. Default "50 100 150 500 1000".
-b: Message sizes in bytes. You can give multiple options to specify multiple message sizes. Default "50 1024 10240".
-s: Backend Sleep Times in milliseconds. You can give multiple options to specify multiple sleep times. Default "0 30 500 1000".
-m: Application heap memory sizes. You can give multiple options to specify multiple heap memory sizes. Default "2g".
-d: Test Duration in seconds. Default 900.
-w: Warm-up time in minutes. Default 5.
-n: Number of JMeter servers. If n=1, only client will be used. If n > 1, remote JMeter servers will be used. Default 1.
-j: Heap Size of JMeter Server. Default 4g.
-k: Heap Size of JMeter Client. Default 2g.
-l: Heap Size of Netty Service. Default 4g.
-i: Scenario name to to be included. You can give multiple options to filter scenarios.
-e: Scenario name to to be excluded. You can give multiple options to filter scenarios.
-t: Estimate time without executing tests.
-p: Estimated processing time in between tests in seconds. Default 60.
-h: Display this help and exit.
```

#### Creating a summary

Use `create-summary-csv.sh` to create a summary CSV file.

```console
ubuntu@server:~$ ./jmeter/create-summary-csv.sh -h

Usage: 
./jmeter/create-summary-csv.sh -n <application_name> [-p <file_prefix>] [-g <gcviewer_jar_path>] [-d <results_dir>]
   [-j <jmeter_servers>] [-w] [-i] [-h]

-n: Name of the application to be used in column headers.
 p: Prefix of the files to get metrics (Load Average, GC, etc).
-g: Path of GCViewer Jar file, which will be used to analyze GC logs.
-d: Results directory. Default ./jmeter/results.
-j: Number of JMeter servers. If n=1, only client was used. If n > 1, remote JMeter servers were used. Default 1.
-w: Use warmup results instead of measurement results.
-i: Include GC statistics and load averages for other servers.
-h: Display this help and exit.
```

Use `csv-to-markdown-converter.py` to convert CSV results into Markdown format.

```console
ubuntu@server:~$ ./jmeter/csv-to-markdown-converter.py

Usage: {Input File(.csv)} {Output File (.md)}
```

### JTL Splitter

The "jtl-splitter" directory has a Java program to split a single JTL file into warmup and measurement based on the 
 number of minutes given as the warmup time.

When reporting the results for the performance tests, some specified number of minutes from the beginning of the test 
 are considered as the "Java Warm-up Time" and the from the final results, the warm-up duration is excluded. 
 By doing this, the results reported from the test will only consider the steady-state of the server.

This program should be invoked by the performance testing script after completing the JMeter performance test.

For example if you specify 5 minutes warmup-time, the JTL splitter splits the `results.jtl` file and the `results-warmup.jtl`
file will have the test results for first 5 minutes. The results after 5 minutes will be in `results-measurement.jtl`.

```console
ubuntu@server:~$ ./jtl-splitter/jtl-splitter.sh -h

Usage: 
./jtl-splitter/jtl-splitter.sh [-m <heap_size>] [-h] -- [jtl_splitter_flags]

-m: The heap memory size. Default: 1g
-h: Display this help and exit.
```

JTL Splitter usage:

```console
ubuntu@server:~$ ./jtl-splitter/jtl-splitter.sh -- -h
Usage: JTLSplitter [options]
  Options:
    -d, --delete-jtl-file-on-exit
      Delete JTL File on exit
      Default: false
    -h, --help
      Display Help
  * -f, --jtlfile
      JTL File
    -n, --precision
      Precision to use in statistics
      Default: 2
    -p, --progress
      Show progress
      Default: false
    -s, --summarize
      Summarize results
      Default: false
    -u, --time-unit
      Time Unit
      Default: MINUTES
      Possible Values: [NANOSECONDS, MICROSECONDS, MILLISECONDS, SECONDS, MINUTES, HOURS, DAYS]
  * -t, --warmup-time
      Warmup Time
      Default: 0
```


### Netty Service

The "netty-service" directory has a simple Netty HTTP Echo Service, which will echo back the body data in the HTTP 
 request.

The Netty HTTP Echo Service should be started by the performance testing script.

```console
ubuntu@server:~$ ./netty-service/netty-start.sh -h

Usage: 
./netty-service/netty-start.sh [-m <heap_size>] [-h] -- [netty_service_flags]

-m: The heap memory size of Netty Service. Default: 4g
-h: Display this help and exit.
```

The script also accepts an argument to specify the number of milliseconds to sleep before sending response. This is
 useful to test the performance with delays.

```console
ubuntu@server:~$ ./netty-service/netty-start.sh -- -h
Starting Netty
Usage: EchoHttpServer [options]
  Options:
    --boss-threads
      Boss Threads
      Default: 4
    --enable-ssl
      Enable SSL
      Default: false
    -h, --help
      Display Help
    --port
      Server Port
      Default: 8688
    --sleep-time
      Sleep Time in milliseconds
      Default: 0
    --worker-threads
      Worker Threads
      Default: 200
```

### Payloads

The "payloads" directory has a Java program to generate JSON payloads with different sizes.

By default, the script generates 50B, 1KiB, 10KiB, and 100KiB JSON files.

If you want to generate different payload sizes, pass the payload sizes as parameters.

The performance testing script can call this script to generate payloads required for the performance test.

```console
ubuntu@server:~$ ./payloads/generate-payloads.sh -h

Usage: 
./payloads/generate-payloads.sh [-p <payload_type>] [-s <payload_size>]

-p: The Payload Type.
-s: The Payload Size. You can give multiple payload sizes.
-h: Display this help and exit.
```

### SAR

The "sar" directory has a simple script to install System Activity Report (SAR) in Linux and configure it to run every
 one minute.

The script needs to be run as root.

```console
ubuntu@server:~$ sudo ./sar/install-sar.sh -h

Usage: 
./sar/install-sar.sh [-h]

-h: Display this help and exit.
```

### Setup scripts

The "setup" directory has the scripts to setup instances (for example, JMeter Client, JMeter Server, Netty Server, etc.)

#### Common Setup Script

The `setup-common.sh` script is used by all other setup scripts to do some common operations.

```console
ubuntu@server:~$ sudo ./setup/setup-common.sh -h

Usage: 
./setup/setup-common.sh  [-g] [-p <package>] [-w <url_to_download>] [-o <output_name>]

-g: Upgrade distribution
-p: Package to install. You can give multiple -p options.
-w: Download URLs. You can give multiple URLs to download.
-o: Output name of the downloaded file. You can give multiple names for a given set of URLs.
-h: Display this help and exit.
```

#### Setup JMeter

The `setup-jmeter.sh` installs JMeter and JMeter plugins. This script uses the `install-jmeter.sh` script in "jmeter"
directory.

```console
ubuntu@server:~$ sudo ./setup/setup-jmeter.sh -h

Usage: 
./setup/setup-jmeter.sh -i <installation_dir> [-j <jmeter_plugin>]  [-g] [-p <package>] [-w <url_to_download>] [-o <output_name>]

-g: Upgrade distribution
-p: Package to install. You can give multiple -p options.
-w: Download URLs. You can give multiple URLs to download.
-o: Output name of the downloaded file. You can give multiple names for a given set of URLs.
-i: The JMeter installation directory.
-j: The JMeter plugin name. You can give multiple JMeter plugins to install.
-h: Display this help and exit.
```

#### Setup JMeter Client

The `setup-jmeter-client.sh` uses `setup-jmeter.sh` internally. It also creates the SSH configurations to execute commands
in other instances.

```console
ubuntu@server:~$ sudo ./setup/setup-jmeter-client.sh -h

Usage: 
./setup/setup-jmeter-client.sh -k <key_file> -i <installation_dir> -c <ssh_config_location> -a <ssh_alias> -n <ssh_hostname> [-j <jmeter_plugin>]  [-g] [-p <package>] [-w <url_to_download>] [-o <output_name>]

-g: Upgrade distribution
-p: Package to install. You can give multiple -p options.
-w: Download URLs. You can give multiple URLs to download.
-o: Output name of the downloaded file. You can give multiple names for a given set of URLs.
-k: The key file location.
-i: The JMeter installation directory.
-c: The SSH config location.
-a: SSH Alias. You can give multiple ssh aliases.
-n: SSH Hostname. You can give multiple ssh hostnames for a given set of ssh aliases.
-j: The JMeter plugin name. You can give multiple JMeter plugins to install.
-h: Display this help and exit.
```

#### Setup Netty Server

The `setup-netty.sh` installs Netty Server and OpenJDK

```console
ubuntu@server:~$ sudo ./setup/setup-netty.sh -h

Usage: 
./setup/setup-netty.sh  [-g] [-p <package>] [-w <url_to_download>] [-o <output_name>]

-g: Upgrade distribution
-p: Package to install. You can give multiple -p options.
-w: Download URLs. You can give multiple URLs to download.
-o: Output name of the downloaded file. You can give multiple names for a given set of URLs.
-h: Display this help and exit.
```

## License

Copyright 2017 WSO2 Inc. (http://wso2.com)

Licensed under the Apache License, Version 2.0
