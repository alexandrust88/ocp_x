# 1. Setting up Prometheus

## Installation

## Setup

Let’s begin with the installation of Prometheus by downloading and extracting the Prometheus binary.

1. Open a new terminal, navigate to your home directory and create the directories work and downloads:

`mkdir ~/{work,downloads}`

`cd ~/downloads`

2. Download Prometheus:

`curl -L -O https://github.com/prometheus/prometheus/releases/download/v2.39.0/prometheus-2.39.0.linux-amd64.tar.gz`

Note: Binaries for other CPU architectures such as ARM or other operating systems (e.g., Darwin, BSD, Windows) are available on the release page of Prometheus: https://github.com/prometheus/prometheus/releases

1. Extract the archive to the work folder:

`tar fvxz prometheus-2.39.0.linux-amd64.tar.gz -C ~/work`

Note: In theory, we could simply run Prometheus by executing the prometheus binary in ~/work/prometheus-2.39.0.linux-amd64. However, to simplify tasks such as reloading or restarting, we are going to create a systemd unit file.

1. Copy the prometheus and promtool binaries to /usr/local/bin

`sudo cp ~/work/prometheus-2.39.0.linux-amd64/{prometheus,promtool} /usr/local/bin`

2. Create the systemd unit file and reload systemd manager configuration

`sudo curl -o /etc/systemd/system/prometheus.service https://raw.githubusercontent.com/puzzle/prometheus-training/main/`

`content/en/docs/01/labs/prometheus.service`
`sudo systemctl daemon-reload`

3. Create the required directories for Prometheus

`sudo mkdir /etc/prometheus /var/lib/prometheus`

`sudo chown ansible.ansible /etc/prometheus /var/lib/prometheus /etc/systemd/system/prometheus.service`

`sudo chmod g+w /etc/prometheus /var/lib/prometheus /etc/systemd/system/prometheus.service`

4. Copy the Prometheus configuration to /etc/prometheus/prometheus.yml

`cp ~/work/prometheus-2.39.0.linux-amd64/prometheus.yml /etc/prometheus/prometheus.yml`

## Configuration

The configuration of Prometheus is done using a YAML config file and CLI flags. The Prometheus tarball we downloaded earlier includes a very basic example of a Prometheus configuration file:

`/etc/prometheus/prometheus.yml`

>
    # my global config
    global:
    scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
    evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # scrape_timeout is set to the global default (10s).

    # Alertmanager configuration
    alerting:
    alertmanagers:
        - static_configs:
            - targets:
            # - alertmanager:9093

    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
    # - "first_rules.yml"
    # - "second_rules.yml"

    # A scrape configuration containing exactly one endpoint to scrape:
    # Here it's Prometheus itself.
    scrape_configs:
    # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
    - job_name: "prometheus"

        # metrics_path defaults to '/metrics'
        # scheme defaults to 'http'.

        static_configs:
        - targets: ["localhost:9090"]

Let’s take a look at two important configuration options:
> - scrape_interval: Prometheus is a pull-based monitoring system which means it will reach out to the configured targets and collect the metrics from them (instead of a push-based approach where the targets will push their metrics to the monitoring server). The option scrape_interval defines the interval at which Prometheus will collect the metrics for each target.
> - scrape_configs: This block defines which targets Prometheus will scrape. In the configuration above, only a single target (the Prometheus server itself at localhost:9090) is configured. Check out the targets section below for a detailed explanation.

Note: We will learn more about other configuration options (evaluation_interval, alerting, and rule_files) later in this training.

## Run Prometheus

1. Start Prometheus and verify

`sudo systemctl start prometheus`

2. Verify that Prometheus is up and running by navigating to http://localhost:9090 with your browser. You should now see the Prometheus web UI.

## Targets

Since Prometheus is a pull-based monitoring system, the Prometheus server maintains a set of targets to scrape. This set can be configured using the scrape_configs option in the Prometheus configuration file. The scrape_configs consist of a list of jobs defining the targets as well as additional parameters (path, port, authentication, etc.) which are required to scrape these targets.

Note: Each job definition must at least consist of a job_name and a target configuration (i.e., static_configs). Check the Prometheus docs for the list of all available options in the scrape_config.
There are two basic types of target configurations:

## Static configuration (example)

In this case, the Prometheus configuration file contains a static list of targets. In order to make changes to the list, you need to change the configuration file. We used this type of configuration in the previous section to scrape the metrics of the Prometheus server:

>
    ...
    scrape_configs:
    ...
    - job_name: "example-job" # this is a minimal example of a job definition containing the job_name and a target configuration
        static_configs:
        - targets:
        - server1:8080
        - server2:8080
    ...

## Dynamic configuration (example)

In addition to the static target configuration, Prometheus provides many ways to dynamically add/remove targets. There are builtin service discovery mechanisms for cloud providers such as AWS, GCP, Hetzner, and many more. In addition, there are more versatile discovery mechanisms available which allow you to implement Prometheus in your environment (e.g., DNS service discovery or file service discovery). Let’s take a look at an example of a file service discovery configuration:

>
    ...
    scrape_configs:
    ...
    - job_name: example_file_sd
        file_sd_configs:
        - files:
        - /etc/prometheus/file_sd/targets.yml
    ...

In this example, Prometheus will lookup a list of targets in the file /etc/prometheus/file_sd/targets.yml. Prometheus will also pickup changes in the file automatically (without reloading) and adjust the list of targets accordingly.

## Relabeling (advanced)

Relabeling in Prometheus can be used to perform numerous tasks using regular expressions, such as
>- adding, modifying or removing labels to/from metrics or alerts,
> - filtering metrics based on labels, or
> -  enabling horizontal scaling of Prometheus by using hashmod relabeling.

It is a very powerful part of the Prometheus configuration, but it can also get quite complex and confusing. Thus, we will only take a look at some basic/simple examples.

There are four types of relabelings:
> - relabel_configs (target relabeling)

Target relabeling is defined in the job definition of a scrape_config. This is used to configure scraping of a multi-target exporter (e.g., blackbox_exporter or snmp_exporter) where one single exporter instance is used to scrape multiple targets. Check out the Prometheus docs for a detailed explanation and example configurations of relabel_configs.

> - metric_relabel_configs (metrics relabeling)

Metrics relabeling is applied to scraped samples right before ingestion. It allows adding, modifying, or dropping labels or even dropping entire samples if they match certain criteria.

> - alert_relabel_configs (alert relabeling)

Alert relabeling is similar to metric_relabel_configs, but applies to outgoing alerts.

> - write_relabel_configs (remote write relabeling)

Remote write relabeling is similar to metric_relabel_configs, but applies to remote_write configurations.

***

# 1.1 Tasks: Setup

In this first lab you are going to configure Prometheus to scrape the node_exporter.

Task 1.1: Node exporter
node_exporter is a Prometheus exporter for hardware and OS metrics. Or in other words, it supplies us with the more common metrics we know from classic monitoring systems. It is therefore very useful for expanding Prometheus’ monitoring capabilities into the infrastructure world. node_exporter is already installed on your system and can be controlled using systemctl.

Make sure node_exporter is running and available at the following endpoint: http://localhost:9100

So first of all, we are going make sure that node_exporter is running:

`sudo systemctl status node_exporter.service`

Test if the node_exporter works correctly:

`curl http://localhost:9100`

The above command should output this HTML page:

>
    <html>
    <head><title>Node Exporter</title></head>
    <body>
    <h1>Node Exporter</h1>
    <p><a href="/metrics">Metrics</a></p>
    </body>
    </html>

If this is the case, node_exporter is working correctly in your environment!

## Task 1.2: Scrape configuration

The node_exporter is now running and exposes hardware and OS metrics. If you’d like to check this, check node_exporter’s endpoint again but this time, attach the path /metrics:

`curl http://localhost:9100/metrics`

What you will see if you do this is a huge bunch of different values and identifiers. The values represent the actual resource consumption on your system like, e.g., consumed memory. In order to make full use and put these metrics into perspective, we need to get them into Prometheus. We need Prometheus to scrape them.

Scraping means that Prometheus regularly collects metrics from all configured endpoints, like node_exporter’s endpoint we just set up, parses them and saves them in its time series database. And this is what we are going to do next.

Configure Prometheus to scrape the metrics from node_exporter.

There are two different ways to achieve that: Either we configure Prometheus using a static configuration or we make use of Prometheus’ service discovery mechanism.

## Option 1: Static configuration

If you decide to configure Prometheus statically, this is what it looks like:

>
    global:
    scrape_interval: 15s
    evaluation_interval: 15s

    alerting:
    alertmanagers:
        - static_configs:
            - targets:
            # - alertmanager:9093

    rule_files:

    scrape_configs:
    - job_name: "prometheus"
        static_configs:
        - targets: ["localhost:9090"]

    - job_name: "node_exporter"
        static_configs:
        - targets: ["localhost:9100"]


What’s new here is the node_exporter part under scrape_configs. It adds node_exporter’s endpoint as a target to Prometheus.

Open your Prometheus configuration (/etc/prometheus/prometheus.yml) and add the new target.

## Option 2: Service discovery

Note: Either apply option 1 or 2, not both.

If you decide to apply the dynamic solution, you have to adapt two files. One is /etc/prometheus/prometheus.yml:

>
    global:
    scrape_interval:     15s
    evaluation_interval: 15s

    alerting:
    alertmanagers:
    - static_configs:
        - targets:

    rule_files:

    scrape_configs:
    - job_name: "prometheus"
        static_configs:
        - targets: ["localhost:9090"]

    - job_name: "node_exporter"
        file_sd_configs:
        - files:
        - node_exporter_targets.yml

The other one is /etc/prometheus/node_exporter_targets.yml:

>
    - targets:
        - 127.0.0.1:9100

As in the first option, Prometheus’ configuration is extended by a scrape_config. However, it isn’t a target this time but rather a reference to a file that contains the targets.

## Reload Prometheus

Whenever you change its configuration, Prometheus needs to be reloaded to apply these changes. One way to do this is to use systemctl:

`sudo systemctl reload prometheus`

If you started Prometheus with the --web.enable-lifecycle flag it’s also possible to reload the Prometheus configuration using a POST request:

`curl -X POST http://localhost:9090/-/reload`

Note: Please note that --web.enable-lifecycle allows anyone with access to the Prometheus HTTP API to reload and even terminate Prometheus using the /-/quit endpoint.

### Verify

After configuring the new target and reloading Prometheus, node_exporter should show up in the list of targets .

*Hint: Sometimes it is helpful to check if the config files are valid. You can do this with the following command. Any errors are indicated including the line number.*

`promtool check config /etc/prometheus/prometheus.yml`

## Task 1.3: Further configuration (optional)

This task is based on the metric net_conntrack_dialer_conn_failed_total and consists of two parts.
> - Check metrics for net_conntrack_dialer_conn_failed_total
> - Use a metric relabel configuration to drop the metric net_conntrack_dialer_conn_failed_total

Verify the metric in the Prometheus we UI .

Alter the existing prometheus job configuration in /etc/prometheus/prometheus.yml

>
    ...
    - job_name: "prometheus"

        # metrics_path defaults to '/metrics'
        # scheme defaults to 'http'.

        static_configs:
        - targets: ["localhost:9090"]
        metric_relabel_configs:
        - source_labels: [ __name__ ]
            regex: "net_conntrack_dialer_conn_failed_total"
            action: drop
    ...

Reload the configuration:

`sudo systemctl reload prometheus`

The metric still gets exposed on the exporter side, but is no longer available to query in the Prometheus User Interface as it gets dropped during scrape time.

`curl localhost:9090/metrics | grep net_conntrack_dialer_conn_failed_total`

Note: Dropping metrics should be a last resort when it is not possible to disable metrics in the first place on the exporter side. For example, you can define collectors for the Prometheus Node Exporter and specify granularly which metrics should be available.

Please note that due to the so-called staleness , it may take up to 5 minutes until changes show up.

***

# 2. Metrics

## Prometheus exposition format

Note: Prometheus consumes metrics in Prometheus text-based exposition format and plans to adopt the OpenMetrics standard: https://prometheus.io/docs/introduction/roadmap/#adopt-openmetrics .

Prometheus Exposition Format
>
    # HELP <metric name> <info>
    # TYPE <metric name> <metric type>
    <metric name>{<label name>=<label value>, ...} <sample value>

## FIXME web UI

As an example, check the metrics of your Prometheus server (http://localhost:9090/metrics ).

>
    ...
    # HELP prometheus_tsdb_head_samples_appended_total Total number of appended samples.
    # TYPE prometheus_tsdb_head_samples_appended_total counter
    prometheus_tsdb_head_samples_appended_total 463
    # HELP prometheus_tsdb_head_series Total number of series in the head block.
    # TYPE prometheus_tsdb_head_series gauge
    prometheus_tsdb_head_series 463
    ...

Note: There are 4 different metric types in Prometheus

> - Counter
> - Gauge
> - Histogram
> - Summary

*Prometheus Metric Types*

## Explore Prometheus metrics

**FIXME web UI**

Open your Prometheus web UI and navigate to the Graph menu. You can use the Open metrics explorer icon (next to the Execute button) to browse your metrics or start typing keywords in the expression field. Prometheus will try to find metrics that match your text.

Learn more about:

> - Prometheus operators
> - Prometheus functions
> - PromLens , the power tool for querying Prometheus

## Recording Rules

Prometheus recording rules allow you to precompute queries at a defined interval (global.evaluation_interval or interval in rule_group) and save them to a new set of time series.

## Special labels

As you have already seen in several examples, a Prometheus metric is defined by one or more labels with the corresponding values. Two of those labels are special, because the Prometheus server will automatically generate them for every metric:

> - instance

The instance label describes the endpoint where Prometheus scraped the metric. This can be any application or exporter. In addition to the ip address or hostname, this label usually also contains the port number. Example: 10.0.0.25:9100

> - job

This label contains the name of the scrape job as configured in the Prometheus configuration file. All instances configured in the same scrape job will share the same job label.

Note: Prometheus will append these labels dynamically before sample ingestion. Therefore you will not see these labels if you query the metrics endpoint directly (e.g. by using curl).
Let’s take a look at the following scrape config (example, no need to change the Prometheus configuration on your lab VM):

> 
    ...
    scrape_configs:
    ...
    - job_name: "node_exporter"
        static_configs:
        - targets:
            - "10.0.0.25:9100"
            - "10.0.0.26:9100"
            - "10.0.0.27:9100"
    ...

In the example above we configured a single scrape job with the name node_exporter and three targets. After ingestion into Prometheus, every metric scraped by this job will have the label: job="node_exporter". In addition, metrics scraped by this job from the target 10.0.0.25 will have the label instance="10.0.0.25:9100"

***

# 2.1 Tasks: Explore metrics

In this lab you are going to explore various metrics, which your Prometheus server is collecting.

## Task 2.1.1: Prometheus web UI

Get a feel for how to use the Prometheus web UI. Open the web UI and navigate to the Graph menu (right on top in the grey navigation bar next to Alerts).

### FIXME web UI

Open the [web UI](FIXME: point to Thanos Querier) and navigate to the Graph menu (right on top in the grey navigation bar next to Alerts).

![F](F1.png)

Let’s start and find a memory related metric. The best way to start is by typing node_memory in the expression bar.

Note: As soon as you start typing a dropdown with matching metrics is shown.

You can also open the Metrics Explorer by clicking on the globe symbol next to the Execute button.

Select a metric such as node_memory_MemFree_bytes and click the Execute button.

The result of your first Query will be available under the two tabs:

1. Table
2. Graph

Explore those two views on your results. Shrink the time range in the Graph tab.

## Task 2.1.2: Metric Prometheus server version

Prometheus collects its own metrics, so information such as the current build version of your Prometheus server is displayed as a metric.

Let’s find a metric that shows you the version of your Prometheus server.

Start typing prometheus_... in the expression browser, choose the prometheus_build_info metric and click the Execute Button.

Something similar to the following will be displayed

>
    metricname                                  Value
    prometheus_build_info{branch="HEAD", goversion="go1.19.1", instance="localhost:9090", job="prometheus", revision="6d7f26c46ff70286944991f95d791dff03174eea", version="2.39.0"}

The actual Version of your Prometheus Server will be available as label version

    {version="2.39.0"}

## Task 2.1.3: Metric TCP sockets

Let’s explore a node exporter metric in this lab.

1. Find a metric that shows you the number of TCP sockets in use

2. Display the number 5 minutes ago

3. Display the numbers in a graph over the last 15 minutes

The node exporter metrics are all available in the node namespace .

The number of TCP sockets in use are available in the following metric.

`node_sockstat_TCP_inuse`

If you want to display the value 5 minutes ago, you’ll have to add the correct timestamp in the Evaluation time field.

Switch to the Graph tab and change the value of the timepicker from 1h to 15m to display the graph over the last 15 minutes.

## Task 2.1.4: Metric network interfaces

Most virtual Linux machines nowadays have network interfaces. The node exporter you have enabled and configured in the previous lab also exposes metrics about network components.

Show all network interfaces where the device name starts with ens

The network interfaces are available in the following series:

`node_network_info`

The result includes all sorts of network interface. If you need to filter the result by a label you will have to alter your query:

`node_network_info{device="ens3"}`


But this will only show results for the exact ens3 interface. The Task was to show all interfaces that start with ens.

In this case we have to use Time series Selectors to create a matching filter:

`node_network_info{device=~"ens.*"}`


There will be a lot more about queries and filtering in the next Labs

***

# 2.2 Tasks: PromQL
In this lab you are going to learn a bit more about PromQL (Prometheus Query Language) .

PromQL is the query language that allows you to select, aggregate and filter the time series data collected by prometheus in real time.

Note: PromQL can seem overwhelming. It may take a little time to get used to it. There may be different approaches to solve the tasks. Our solution is just one possibility.

## Task 2.2.1: Explore Examples

In this first task you are going to explore some querying examples.

Get all time series with the metric prometheus_http_requests_total.

`prometheus_http_requests_total`

The result represents the time series for the http requests sent to your Prometheus server as a instant vector.

Get all time series with the metric prometheus_http_requests_total and the given code and handler labels.

`prometheus_http_requests_total{code="200", handler="/api/v1/query"}`

The result will show you the time series for the http requests sent to the query endpoint of your Prometheus Server, which were successful ( HTTP status code 200 ).

Get a whole range of time (5 minutes) for the same vector, making it a range vector:

`prometheus_http_requests_total{code="200", handler="/api/v1/query"}[5m]`

A range vector can not be graphed directly in the Prometheus UI, use the table view to display the result.

With regular expressions you can filter time series only for handlers whose name matches a certain pattern, in this case all handlers starting with /api:

`prometheus_http_requests_total{handler=~"/api.*"}`


All regular expressions in Prometheus use the RE2 syntax . To select all HTTP status codes except 2xx, you would execute:

`prometheus_http_requests_total{code!~"2.."}`

## Task 2.2.2: Sum Aggregation Operator

The Prometheus Aggregation operators help us to aggregate time series in PromQL.

There is a Prometheus metric that represents all samples scraped by Prometheus. Let’s sum up the metrics returned.

The metric scrape_samples_scraped represents the total of scraped samples by job and instance. To get the total amount of scrapped samples, we use the Prometheus aggregation operators sum to sum the values.

`sum(scrape_samples_scraped)`

## Task 2.2.3: Rate Function

Use the rate() function to display the current CPU idle usage per CPU core of the server in % based on data of the last 5 minutes.

*Hint: Read the documentation about the rate() function.*

The CPU metrics are collected and exposed by the node_exporter therefore the metric we’re looking for is under the node namespace.

`node_cpu_seconds_total`

To get the idle CPU seconds, we add the label filter {mode="idle"}.

Since the rate function calculates the per-second average increase of the time series in a range vector, we have to pass a range vector to the function node_cpu_seconds_total{mode="idle"}[5m]

To get the idle usage in % we therefore have to multiply it with 100.

`rate(`

`  node_cpu_seconds_total{mode="idle"}[5m]`

`  )`

`* 100`


## Task 2.2.4: Arithmetic Binary Operator

In the previous lab, we created a query that returns the CPU idle usage. Now let’s reuse that query to create a query that returns the current CPU usage per core of the server in %. The usage is the total (100%) minus the CPU usage idle.

To get the CPU usage we can simply substract idle CPU usage from 1 (100%) and then multiply it by 100 to get percentage.
>
    (
    1 -
    rate(
        node_cpu_seconds_total{mode="idle"}[5m]
        )
    )
    * 100


## Task 2.2.5: How much free memory

Arithmetic Binary Operator can not only be used with constant values eg. 1, it can also be used to evaluate to other instant vectors.

Write a Query that returns how much of the memory is free in %.

The node exporter exposes these two metrics:

> - node_memory_MemTotal_bytes
> - node_memory_MemAvailable_bytes

We can simply divide the available memory metric by the total memory of the node and multiply it by 100 to get percent.

>
    sum by(instance) (node_memory_MemAvailable_bytes)
    /
    sum by(instance) (node_memory_MemTotal_bytes)
    * 100

## Task 2.2.6: Comparison Binary Operators

In addition to the Arithmetic Binary Operator, PromQL also provides a set of Comparison binary operators


    >== (equal)
    != (not-equal)
    > (greater-than)
    < (less-than)
    >= (greater-or-equal)
    <= (less-or-equal)

Check if the server has more than 20% memory available using a Comparison binary operators

We can simply use the greater-than-binary operator to compare the instant vector from the query with 20 (In our case, this corresponds to 20% memory usage).

>
    sum by(instance) (node_memory_MemAvailable_bytes)
    /
    sum by(instance) (node_memory_MemTotal_bytes)
    * 100
    > 20


The query only has a result when more than 20% of the memory is available.

Change the value from 20 to 90 or more to see the result, when the operator doesn’t match.

## Task 2.2.7: Histogram

So far we’ve using gauge and counter metric types in our queries.

Read the documentation about the histogram metric type.

There exists a histogram for the http request durations to the Prometheus sever. It basically counts requests that took a certain amount of time and puts them into matching buckets (le label).

We want to write a query that returns

> - the total numbers of requests
> - to the Prometheus server
> - on /metrics
> - below 0.1 seconds

As seen in previous labs, the http metrics for the Prometheus server are available in the prometheus namespace.

By filtering the le label to 0.1 we get the result for our query.

`prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="0.1"}`

*Tip: Analyze the query in PromLens*

**Advanced: You can calculate how many requests in % were below 0.1 seconds by aggregating above metric. See more information about Apdex score at Prometheus documentation**

Example
>
    sum(
    rate(
        prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="0.1"}[5m]
    )
    ) by (job, handler)
    /
    sum(
    rate(
        prometheus_http_request_duration_seconds_count{handler="/metrics"}[5m]
    )
    ) by (job, handler)
    * 100


## Task 2.2.8: Quantile

We can use the histogram_quantile function to calculate the request duration quantile of the requests to the Prometheus server from a histogram metric. To archive this we can use the metric prometheus_http_request_duration_seconds_bucket, which the Prometheus server exposes by default.

Write a query, that returns the per-second average of the 0.9th quantile under the metrics handler using the metric mentioned above.

Expression

>
    histogram_quantile(
    0.9,
    rate(
        prometheus_http_request_duration_seconds_bucket{handler="/metrics"}[5m]
    )
    )

Explanation: histogram_quantile will calculate the 0.9 quantile based on the samples distribution in our buckets by assuming a linear distribution within a bucket.

## Task 2.2.9: predict_linear function (optional)

We could simply alert on static thresholds. For example, notify when the file system is more than 90% full. But sometimes 90% disk usage is a desired state. For example, if our volume is very large. (e.g. 10% of 10TB would still be 1TB free, who wants to waste that space?) So it is better to write queries based on predictions. Say, a query that tells me that my disk will be full within the next 24 hours if the growth rate is the same as the last 6 hours.

Let’s write a query, that exactly makes such predictions:

> - Find a metric that displays you the available disk space on filesystem mounted on /
> - Use a function that allows you to predict when the filesystem will be full in 4 hours
> - Predict the usage linearly based on the growth over the last 1 hour

Expression

`predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 3600 * 4) < 0`


Explanation: based on data over the last 1h, the disk will be < 0 bytes in 3600 * 4 seconds. The query will return no data because the file system will not be full in the next 4 hours. You can check how much disk space will be available in 4 hours by removing the < 0 part.

`predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 3600 * 4)`

## Task 2.2.10: Many-to-one vector matches (optional)

Prometheus provides built-in metrics that can be used to correlate their values with metrics exposed by your exporters. One such metric is date(). Prometheus also allows you to add more labels from different metrics if you can correlate both metrics by labels. See Many-to-one and one-to-many vector matches for more examples.

Write a query that answers the following questions:

> - What is the uptime of the server in minutes?
> - Which kernel is currently active?

Expression

>
    (
    (
        time() - node_boot_time_seconds
    ) / 60
    )
    * on(instance) group_left(release) node_uname_info


> - time(): Use the current UNIX Epoch time
> - node_boot_time_seconds: Returns the UNIX epoch time at which the VM was started
> - on(instance) group_left(release) node_uname_info: Group your metrics result with the metric node_uname_info which contains information about your kernel in the release label.

Alternative solution with group_right instead of group_left would be:

>
    node_uname_info * on(instance) group_right(release)
    (
    (
        time() - node_boot_time_seconds
    ) / 60
    )

***

# 2.3 Tasks: Recording rules
In this lab you are going to create your first own recording rules. Recoding rules are very useful when it comes to queries, which are very complex and take a long time to compute.

**Warning: Recording rules store the result in a new series and they can add additional complexity.**

## Task 2.3.1: Memory usage recording rule

In this task you’re going to create your first recording rule.

With this recording rule, we create a new metric that represents the available memory on a node as a percentage. A metric the node exporter doesn’t expose when running on a machine with an older Linux kernel and needs to be calculated every time.

> - Query the recording rule in the Prometheus web UI
> - Add the following recording rule in a file named /etc/prometheus/recording_rules.yml and include it in the 

Prometheus config

>
    groups:
    - name: node_memory
        rules:
        - record: :node_memory_MemAvailable_bytes:sum
            expr: |
            (1 - (
                sum by(instance) (node_memory_MemFree_bytes + node_memory_Cached_bytes + node_memory_Buffers_bytes)
            )
            /
                sum by(instance) (node_memory_MemTotal_bytes))
            * 100          


Create the recording rule recording_rules.yml with the following command:

`curl -o /etc/prometheus/recording_rules.yml \`

`https://raw.githubusercontent.com/puzzle/prometheus-training/main/content/en/docs/02/labs/recording_rules.yml`


We also need to tell Prometheus to consider the new recording_rules.yml file, by altering the /etc/prometheus/prometheus.yml.

>
    ...
    rule_files:
    - "recording_rules.yml"
    ...


We can validate the configuration.

`promtool check config /etc/prometheus/prometheus.yml`


The last thing we need to do is to reload Prometheus.

`sudo systemctl reload prometheus`


After configuring the recording rule and reloading the configuration, Prometheus provides those metrics accordingly.

Note: It may take up to one minute for the recording rule to become available.

Use your recording_rule definition in the expression browser:

`:node_memory_MemAvailable_bytes:sum`

FIXME: replace link

Note: If you take a look at the historical metrics, you will notice that there is no backfilling of your data. Only data since activation of the recording rule is available.

## Task 2.3.2: CPU utilization recording rule

In this lab you are going to create a CPU utilization recording rule.

> - Create a rule to record the CPU utilization of your server
> - Make sure that Prometheus evaluates this rule every 60 seconds
> - Verify in the web UI that you can query your recording rule

As you saw in a previous exercise, the node_cpu_seconds_total metric contains the CPU utilization of a node. We can use the mode label on this metric to filter for idle cpu time.

All other modes than idle indicate, that the CPU is used. Therefore we can simply subtract the idle percentage from 100 % and get the value we want.

Add the following recording rule to /etc/prometheus/recording_rules.yml:

>
    groups:
    ...
    - name: node_cpu
        interval: 60s
        rules:
        - record: instance:node_cpu_utilisation:rate5m
            expr: |
            100 - (
                avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
                * 100
            )          


By specifying the interval, you can overwrite the default evaluation interval.

Reload Prometheus

`sudo systemctl reload prometheus`

FIXME: replace link

Query your recording rule using the expression browser

***

# 3. Alerting with Alertmanager

## Installation

## Setup

The alertmanager is already installed on your system and can be controlled using systemctl:


`# status`

`sudo systemctl status alertmanager`

`# start`

`sudo systemctl start alertmanager`

`# stop`

`sudo systemctl stop alertmanager`

`# restart`

`sudo systemctl restart alertmanager`

`# reload`

`sudo systemctl reload alertmanager`


The configuration file of alertmanager is located here: /etc/alertmanager/alertmanager.yml

## Configuration

Alertmanager’s configuration is done using a YAML config file and CLI flags. Take a look at the very basic configuration file at /etc/alertmanager/alertmanager.yml:

>
    global:
    resolve_timeout: 5m

    route:
    group_by: ['alertname']
    group_wait: 10s
    group_interval: 10s
    repeat_interval: 1h
    receiver: 'web.hook'
    receivers:
    - name: 'web.hook'
    webhook_configs:
    - url: 'http://127.0.0.1:5001/'
    inhibit_rules:
    - source_match:
        severity: 'critical'
        target_match:
        severity: 'warning'
        equal: ['alertname', 'dev', 'instance']


## Configuration in Alertmanager

There are two main sections for configuring how Alertmanager is dispatching alerts: receivers and routing.

### Receivers

With a receiver , one or more notifications can be defined. There are different types of notifications types, e.g. mail, webhook, or one of the message platforms like Slack or PagerDuty.

### Routing

With routing blocks , a tree of routes and child routes can be defined. Each routing block has a matcher which can match one or several labels of an alert. Per block, one receiver can be specified, or if empty, the default receiver is taken.

### amtool

As routing definitions might be very complex and hard to understand, amtool becomes handy as it helps to test the rules. It can also generate test alerts and has even more useful features. More about this in the labs.

### More (advanced) options

For more insights of the configuration options, study the following resources:

> - Example configuration provided by Alertmanager on GitHub
> - General overview of Alertmanager

## Alert rules in Prometheus

Prometheus alert rules are configured very similarly to recording rules which you got to know earlier in this training . The main difference is that the rule’s expression contains a threshold (e.g., query_expression >= 5) and that an alert is sent to the Alertmanager in case the rule evaluation matches the threshold. An alert rule can be based on a recording rule or be a normal expression query.

Note: Sometimes the community or the maintainer of your Prometheus exporter already provide generic Prometheus alert rules that can be adapted to your needs. For this reason, it makes sense to do some research before writing alerting rules from scratch. Before implementing such a rule, you should always understand and verify the rule. Here are some examples:

> - MySQL: mysqld-mixin
> - Strimzi Kafka Operator: strimzi/strimzi-kafka-operator
> - General rules for Kubernetes: kubernetes-mixin-ruleset
> - General rules for various exporters: samber/awesome-prometheus-alerts

***

# 3.1 Tasks: Enable and configure Alertmanager

## Task 3.1.1: Enable Alertmanager in Prometheus

The Alertmanager instance we installed before must be configured in Prometheus. Open /etc/prometheus/prometheus.yml, add the config below, and reload the Prometheus config with sudo systemctl reload prometheus.service.

>
    ...
    # Alertmanager configuration
    alerting:
    alertmanagers:
        - static_configs:
            - targets:
                - localhost:9093
    ...


## Task 3.1.2: Add Alertmanager as monitoring target

Note: This setup is only suitable for our lab environment. In real life, you must consider how to monitor your monitoring infrastructure: Having an Alertmanager instance as an Alertmanager AND as a target only in the same Prometheus is a bad idea!

This is repetition: The Alertmanager (localhost:9093) also exposes metrics which can be scraped by Prometheus.

Configure the metric endpoint of Alertmanager in Prometheus and check, if the target in Prometheus can be scraped.

Configure a new job under scrape_configs in /etc/prometheus/prometheus.yml:

>
    ...
    - job_name: "alertmanager"
        static_configs:
        - targets: ["localhost:9093"]
    ...


Reload Prometheus

`sudo systemctl reload prometheus`

Check in the Prometheus web UI if the target can be scraped.

## Task 3.1.3: Query an Alertmanager metric

After you add the Alertmanager metrics endpoint, you will have huge bunch of different values and identifiers.

Use curl to get a list of all available metrics and query any one from the Alertmanager.

To find out which metrics are available for one service you might query its metrics endpoint with curl, e.g. for Alertmanager:

`curl localhost:9093/metrics`

Then you get all metrics as follows (shortened), and you can pick whatever you’re interested in.

>
    # HELP alertmanager_alerts How many alerts by state.
    # TYPE alertmanager_alerts gauge
    alertmanager_alerts{state="active"} 0
    alertmanager_alerts{state="suppressed"} 0
    # HELP alertmanager_alerts_invalid_total The total number of received alerts that were invalid.
    # TYPE alertmanager_alerts_invalid_total counter
    alertmanager_alerts_invalid_total{version="v1"} 0
    alertmanager_alerts_invalid_total{version="v2"} 0
    # HELP alertmanager_alerts_received_total The total number of received alerts.
    # TYPE alertmanager_alerts_received_total counter
    alertmanager_alerts_received_total{status="firing",version="v1"} 0
    alertmanager_alerts_received_total{status="firing",version="v2"} 0
    alertmanager_alerts_received_total{status="resolved",version="v1"} 0
    alertmanager_alerts_received_total{status="resolved",version="v2"} 0
    ...


## Task 3.1.4: Get all Metrics from Alertmanager

After you successfully configured Prometheus to scrape the Alertmanager you can also query them using PromQL

Write a PromQL query, which selects all metrics exposed by the Alertmanager (job="alertmanager").

To do that, we can simply execute a query without a metrics name and only the job label filter job="alertmanager".

`{job="alertmanager"}`

***

# 3.2 Tasks: Alertmanager

Note: We provide a training app which simulates a webhook receiver. This simulation just prints the alerts it gets. Before we go on, we must install the training app and start the webook receiver:

`cd ~/downloads`

`curl -L -O https://github.com/acend/prometheus-training-app/releases/download/v0.0.5/prometheus-training-app_0.0.5_Linux_x86_64.tar.gz`

`mkdir ~/work/prometheus-training-app_0.0.5_Linux_x86_64 && tar fvxz prometheus-training-app_0.0.5_Linux_x86_64.tar.gz -C $_`

`cd ~/work/prometheus-training-app_0.0.5_Linux_x86_64`

`./prometheus-training-app webhook &`

The URL of this webhook receiver is http://localhost:5001/.

## Task 3.2.1: Send a test alert

In this taks you can use the amtool command to send a test alert.

To send a test alert with the labels alername=UP and node=bar you can simply execute the following command.

`amtool alert add --alertmanager.url=http://localhost:9093 alertname=Up node=bar`


Check in the Alertmanger web UI if you see the test alert with the correct labels set.

## Task 3.2.2: Add a webhook receiver

You can use the amtool to validate your configuration and show your routing configuration.

Check the configuration:

`amtool check-config /etc/alertmanager/alertmanager.yml`

>
    Checking '/etc/alertmanager/alertmanager.yml'  SUCCESS
    Found:
    - global config
    - route
    - 1 inhibit rules
    - 1 receivers
    - 1 templates
    SUCCESS


Show routing tree:

`amtool config routes --config.file /etc/alertmanager/alertmanager.yml`

>
    Routing tree:
    .
    └── default-route  receiver: web.hook


Task description:

> - Configure an additional receiver receiver-a which notifies webhook http://127.0.0.1:5001
> - Define a route so that receiver-a is notified when the label team=team-a is set
> - Use amtool to test your routing configuration

Add the receiver and the route in /etc/alertmanager/alertmanager.yml:

>
    ...
    receivers:
    - name: web.hook
    webhook_configs:
    - url: http://127.0.0.1:5001/
    - name: receiver-a
    webhook_configs:
    - url: http://127.0.0.1:5001/
    ...
    route:
    ...
    routes:
    - receiver: 'receiver-a'
        matchers:
        - team = "team-a"
    ...


Check that receiver-a gets notified, when the label team=team-a is set:

`amtool config routes test --config.file /etc/alertmanager/alertmanager.yml team=team-a`


`receiver-a`


## Task 3.2.3: Add an additional mail receiver

Alertmanger also supports to send mails. See <email_config> for a configuration example. To troubleshoot your configuration, e.g. if you cannot receive mails, it may be worthwhile to have a look at the Alertmanager logs.

`sudo journalctl -u alertmanager`


Additionally, it is possible to use regex to match labels in the routing configuration. To do so, we need to use matchers instead of the match keyword used in the task above. We recommend that you always use the amtool to test your routing configuration for complex alert routing.

Task description:

> - Configure another receiver receiver-b to send a mail
>> - from prometheus@localhost
>> - to alert@localhost
>> - SMTP Server localhost:1025
>> - disable TLS

> - Define two routes
>> - The first route notifies receiver-a when the label team=team-a is set.
>> - The second route notifies receiver-b when the label team=team-a OR team=team-b is set
>> - Make sure that both receivers (A and B) are notified when the label team=team-a is set

> - Send a test alert and verify the mails on mailcatcher http://127.0.0.1:1080
> - Use amtool with both labels to test your routing configuration

In /etc/alertmanager/alertmanager.yml add the receiver and the routes.

>
    ...
    receivers:
    - name: web.hook
    webhook_configs:
    - url: http://127.0.0.1:5001/
    - name: receiver-a
    webhook_configs:
    - url: http://127.0.0.1:5001/
    - name: receiver-b
    email_configs:
    - to: 'alert@localhost'
        from: 'prometheus@localhost'
        smarthost: 'localhost:1025'
        require_tls: false
    ...
    route:
    ...
    routes:
    - receiver: 'receiver-a'
        matchers:
        - team = "team-a"
        continue: true
    - receiver: 'receiver-b'
        matchers:
        - team =~ "team-[a|b]"
    ...


Reload Alertmanager

`sudo systemctl reload alertmanager.service`

Use again amtool for checking your routing configuration:

`amtool config routes --config.file /etc/alertmanager/alertmanager.yml`

>
    Routing tree:
    .
    └── default-route  receiver: web.hook
        ├── {team="team-a"}  continue: true  receiver: receiver-a
        └── {team=~"team-[a|b]"}  receiver: receiver-b


Add a test alert and check if the mailcatcher received the mail. It can take up to 5 minutes as the alarms are grouped together based on the group_interval .

`amtool alert add --alertmanager.url http://localhost:9093 alertname=test team=team-a`


It is also advisable to validate the routing configuration against a test dataset to avoid unintended changes. With the option --verify.receivers the expected output can be specified:

`amtool config routes test --config.file /etc/alertmanager/alertmanager.yml --verify.receivers=receiver-a team=team-b`


`receiver-b`

**WARNING: Expected receivers did not match resolved receivers.**


`amtool config routes test --config.file /etc/alertmanager/alertmanager.yml --verify.receivers=receiver-b team=team-b`


`receiver-b`


## Task 3.2.4: Last successful Alertmanager configuration reload (optional)

Use PromQL to answer the following question: How many minutes ago was the last successful configuration reload of Alertmanager?

`(time() - alertmanager_config_last_reload_success_timestamp_seconds) / 60`


> - time() will return the current UNIX Epoch timestamp
> - alertmanager_config_last_reload_success_timestamp_seconds will return the UNIX Epoch timetamp of the last successful Alertmanager configuration reload
> - /60 will calculate the result in minutes

***

# 3.3 Tasks: Alertrules and alerts

Note: For doing the alerting lab it’s useful to have a “real” application so that alerts can be provoked. The training app installed in the previous lab provides a sample app; you can start it as follows:

`cd ~/work/prometheus-training-app_0.0.5_Linux_x86_64`

`./prometheus-training-app sampleapp &`


The example app exposes metrics at http://localhost:8080/metrics

Also, the target must be registered in the Prometheus config (/etc/prometheus/prometheus.yml) (don’t forget to reload or restart Prometheus):

>
    ...
    - job_name: "sample-app"
        static_configs:
        - targets: ["localhost:8080"]
    ...


## Task 3.3.1: Configure a target down alert

Refer to the official documentation to see which fields you can specify for an alerting rule.

Task description:

> - Define an alerting rule which sends an alert when a target is down. Remember the up metric?
> - New alarms should be in pending state for 2 minutes before they transition to firing
> - Add a label team with the value team-a
> - Add an annotation summary with information about which instance and job is down

Create a new file for Prometheus /etc/prometheus/alertrule.yml and add the following snippet:

>
    groups:
    - name: basic
    rules:
    - alert: Up
        expr: up == 0
        for: 2m
        labels:
        team: team-a
        annotations:
        summary: Instance {{ $labels.instance }} of job {{ $labels.job }} is down


The value in field for is the wait time until the active alert gets in state FIRING. Before that, the alert is PENDING and not yet sent to Alertmanager.

The alert is instrumented with the labels from the metric (e.g. joband instance). Additional labels can be defined in the rule. Labels can be used in Alertmanager for the routing.

With annotations, additional human-readable information can be attached to the alert.

In /etc/prometheus/prometheus.yml, add the rule file at rule_files section and restart or reload Prometheus.

>
    ...
    rule_files:
    - "recording_rules.yml"
    - "alertrule.yml"
    ...


In the Prometheus web UI there is an Alerts menu item which shows you information about the alerts.

## Task 3.3.2: Verify the target down alert

In this task you’re going to explore what happens, when a target (our sample application) that exposes metrics fails and stops working.

> - Simulate a failure by killing or stopping the sample application.
> - Verify that the sample app no longer exposes metrics
> - What do you observe in Prometheus UI or in Alertmanager UI?

You can stop the application by simply killing it:

`pkill -f "prometheus-training-app sampleapp"`


And verify that the sample application doesn’t expose any metrics anymore.

`curl http://localhost:8080/metrics`


    curl: (7) Failed connect to localhost:8080; Connection refused


> - The Prometheus web UI Alerts menu item shows you information about inactive and active alerts.
> - As soon as an alert is in state FIRING the alert is sent to Alertmanager. You should see the alert in its web UI .
> - You can also check the mail in mailcatcher

## Task 3.3.3: Identify the notified receivers

Which receiver was notified when the alarm was fired?

Receivers receiver-a and receiver-b should be notified in this case.

Explanation:

> - The label team: team-a set on the alert rule matches both receivers
> - continue: true is set to true, therefore both receivers will be notified

>
    ...
    routes:
        - receiver: 'receiver-a'
        match:
            team: 'team-a'
        continue: true
        - receiver: 'receiver-b'
        matchers:
            - team =~ "team-[a|b]"
    ...

***

# 4. Prometheus exporters
An increasing number of applications directly instrument a Prometheus metrics endpoint. This enables applications to be scraped by Prometheus out of the box. For all other applications, an additional component (the Prometheus exporter) is needed to close the gap between Prometheus and the application which should be monitored.

Note: There are lots of exporters available for many applications, such as MySQL/MariaDB, Nginx, Ceph, etc. Some of these exporters are maintained by the Prometheus GitHub organization while others are maintained by the community or third-party vendors. Check out the list of exporters on the Prometheus website for an up-to-date list of exporters.
O
ne example of a Prometheus exporter is the node_exporter we used in the first chapter of this training. This exporter collects information from different files and folders (e.g., /proc/net/arp, /proc/sys/fs/file-nr, etc.) and uses this information to create the appropriate Prometheus metrics. In the tasks of this chapter we will configure two additional exporters.

## Special exporters

### Blackbox exporter

This is a classic example of a multi-target exporter which uses relabeling to pass the targets to the exporter. This exporter is capable of probing the following endpoints:

> - HTTP
> - HTTPS
> - DNS
> - TCP
> - ICMP

By using the TCP prober you can create custom checks for almost any service including services using STARTTLS. Check out the example.yml file in the project’s GitHub repository.

### Prometheus Pushgateway

The Pushgateway allows jobs (e.g., Kubernetes Jobs or CronJobs) to push metrics to an exporter where Prometheus will collect them. This can be required since jobs only exist for a short amount of time and as a result, Prometheus would fail to scrape these jobs most of the time. In addition, it would require all these jobs to implement a webserver in order for Prometheus to collect the metrics.

Note: The Pushgateway should only be used for for this specific use case. It simply acts as cache for short-lived jobs and by default does not even have any persistence. It is not intended to convert Prometheus into a push-based monitoring system

***

# 4.1 Tasks: Blackbox exporter

## Task 4.1.1: Add a blackbox target

We will add the pre-installed blackbox exporter to our Prometheus configuration and create a new module which accepts a 418 return code as a valid http return code. This will return the probe_success metric from the blackbox exporter with the value 1, if the http status code is 418.

Task description:

> - Create a new module in the blackbox exporter config (/etc/blackbox_exporter.yml) which uses the HTTP prober and expects a 418 return code as a valid status code
> - Add a job to the Prometheus scrape_configs which scrapes the blackbox exporter using the newly created module
> - Define https://httpstat.us/418 as a single static target, which the blackbox should probe

Note: You need to reload the blackbox exporter and Prometheus after making changes to their configuration files.

`sudo systemctl reload blackbox_exporter`

`sudo systemctl reload prometheus`


To configure the blackbox exporter you have to edit the following file:

`/etc/blackbox_exporter.yml`

>
    modules:
    ...
    http_418:
        prober: http
        http:
        preferred_ip_protocol: ip4
        valid_status_codes:
        - 418
    ...


Like you did for other targets, you have to add a new job to the Prometheus scrape config:

`/etc/prometheus/prometheus.yml:`

>
    scrape_configs:
    ...
    - job_name: 'blackbox'
        metrics_path: /probe #1
        params:
        module: [http_418] #2
        static_configs:
        - targets:
        - https://httpstat.us/418 #3
        relabel_configs:
        - source_labels: [__address__] #4
        target_label: __param_target
        - source_labels: [__param_target] #5
        target_label: instance
        - target_label: __address__ #6
        replacement: 127.0.0.1:9115
    ...


Without the params and relabel_config, the target url would look like this: https://httpstat.us/probe

> - 1: The /metrics metrics_path exposes blackbox internal metrics. The /probe metrics_path will give you metrics about the specified external endpoint
> - 2: Use the http_418 module defined in the blackbox.yml.

The url looks now like this: https://httpstat.us/probe&module=http_418

> - 3: Define the external targets the blackbox exporter should probe
> - 4: The __address__ label holds the values from the in 3 specified targets. We will write these to the __param_target label.

The url look now like this: https://httpstat.us/probe?target=https://httpstat.us/418&module=http_418

> - 5: Now we will write the values from the __param_target label to the instance label. Like this we will later be able to use the label instance to filter our targets in our queries.
> - 6: At last we set the __address__ label to our blackbox exporter address, which will then be used as our blackbox exporter hostname and port

Finally the url looks now like this: http://127.0.0.1:9115/probe?target=https://httpstat.us/418&module=http_418

You can verify this by directly running a curl on this url. The probe_success metric should have the value 1.

`curl "http://127.0.0.1:9115/probe?target=https://httpstat.us/418&module=http_418"`


## Task 4.1.2: Query blackbox metrics

Let’s now create a query which selects all metrics belonging to the blackbox exporter target https://httpstat.us/418 and display them in the Prometheus expression browser.

Due to the relabel config you’ve created in task 4.1.1 the actual target https://httpstat.us/418 will end up in the metric label instance.

Therefore we can select all metrics for the target with the following query:

`{instance="https://httpstat.us/418"}`

or directly navigate to your Prometheus instance

**Warning: In the list of metrics you will find one metric with the name up. In the case of a multi-target exporter such as the blackbox exporter this metric will always be up as long as Prometheus is able to successfully scrape the exporter even if the actual target (website, TCP service, etc.) is down. To monitor the state of the targets always use the probe_success metric.**

## Task 4.1.3 (optional): Add a protocol label to your blackbox target

Add the new label protocol to every blackbox exporter target by updating the relabel config. The new label should contain the protocol (HTTP or HTTPS) extracted from the target URL.

To do this we have to alter the Prometheus configuration /etc/prometheus/prometheus.yml:

>
    scrape_configs:
    ...
    - job_name: 'blackbox'
        metrics_path: /probe
        params:
        module: [http_418] # use the module name defined in the blackbox.yml
        static_configs:
        - targets:
        - https://httpstat.us/418
        relabel_configs:
        - source_labels: [__address__]
        target_label: __param_target
        - source_labels: [__param_target]
        target_label: instance
        - target_label: __address__
        replacement: 127.0.0.1:9115
        - source_labels: [instance] #1
        target_label: protocol #2
        regex: '^(.+):.+' #3
        replacement: $1 #4
    ...


> - 1: Use the value from the label instance. This label contains all targets defined at static_configs.targets
> - 2: We will call the new label protocol
> - 3: Capture the first part of your url until :. In our case https from https://httpstat.us/418
> - 4: Replace target_label value with the regex match from source_labels value

***

# 4.2 Tasks: Pushgateway

## Task 4.2.1 - Install and configure Pushgateway

First we will add the pre-installed Pushgateway (localhost:9091) to our Prometheus instance, later we will use our Pushgateway to add metrics and learn how to remove pushed metrics from the Pushgateway.

Configure Prometheus to scrape the metrics from the Pushgateway

Extend the Prometheus /etc/prometheus/prometheus.yml as you would to add a common exporter.

>
    scrape_configs:
    ...
    - job_name: 'pushgateway'
        honor_labels: true
        static_configs:
        - targets:
        - 'localhost:9091'
    ...


Reload Prometheus to make changes active

`sudo systemctl reload prometheus`


You should now see the default metrics exposed by the Pushgetway with the following query:

`{job="pushgateway"}`


You can also directly navigate to your Prometheus instance

## Task 4.2.2 - Push metrics to Pushgateway

In this task you’re now going to push metrics to the pushgateway. This is what you would normally do, after a cronjob has completed successfully. In order to push metrics to the Pushgateway, you can simple send a HTTP POST or PUT request, with the actual metric we want to push as content, to it.

As you have seen earlier the Pushgateway is available under http://localhost:9091 . When pushing metrics to the Pushgateway you always have to specify the job, therefore the URL Path looks like this

`http://localhost:9091/metrics/job/<JOB_NAME>{/<LABEL_NAME>/<LABEL_VALUE>}`


If for example we want to push the prometheus_training_labs_completed_total metric with the value 4 on the job prometheus_training, we can do that by executing the following command:

`echo "prometheus_training_labs_completed_total 4" | curl --data-binary @- http://localhost:9091/metrics/job/prometheus_training`


Verify the metric in the Prometheus we UI . It may take up to 30s ( Depending on the scrape_interval) to be available in Prometheus.

Note: If you see the labels exported_instance and exported_job in the Prometheus expression browser you did not set honor_labels: true in the Pushgateway scrape configuration.

Push the following metric (notice the instance label) to the Pushgateway and make sure the metric gets scraped by Prometheus

>
    # TYPE some_metric_total counter
    # HELP This is just an example metric.
    some_metric_total{job="prometheus_training",instance="myinstance"} 42


To push a metric to the Pushgateway, which then will be scraped by Prometheus, we can simply use the following curl command. Note the actual content of the HTTP request, is just simply the exact metric we want Prometheus to scrape.

Execute the following command to push the metric to your Pushgateway

`cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/prometheus_training/instance/myinstance`

`# TYPE some_metric_total counter`

`# HELP This is just an example metric.`

`some_metric_total 42`

`EOF`


### Command Explanation

If you are not very familiar with the Linux shell. The above command does the following:

> - the cat command reads the actual metric and pipes it to stdin
> - curl sends a HTTP POST request to the url http://localhost:9091/metrics/job/prometheus_training/instance/myinstance with the –data-binary parameter set to stdin (the actual metric)

Verify the metric in the Prometheus we UI . It may take up to 30s ( Depending on the scrape_interval) to be available in Prometheus.

## Task 4.2.3 - Delete Pushgateway metrics

By sending HTTP delete requests to the same endpoint, we can delete metrics from the Pushgateway.

Note: Metrics pushed to the Pushgateway are not automatically purged until you manually delete them via the API or the process restarts. If you persist the metrics with --persistence.file, you should ensure that you have set up a job that cleans up the metrics on a regular basis.

According to the official Pushgateway documentation you can delete either metrics for specific label combinations (exact match required) or all metrics.

Delete the pushed metrics from the Pushgateway.

Note: Deleting metrics requires the cli option –web.enable-admin-api , which we have enabled for you, but which is disabled by default.

To delete the metrics for the job prometheus_training, you can simply execute the following command:

`curl -X DELETE http://localhost:9091/metrics/job/prometheus_training`


Note: This will delete metrics with the label set {job="prometheus_training"} but not {job="prometheus_training",another_label="value"} since the delete methode requires an exact label match.

With the following command you can delete all metrics:

`curl -X PUT http://localhost:9091/api/v1/admin/wipe`

*** 

# 5. Instrumenting with client libraries
While an exporter is an adapter for your service to adapt a service specific value into a metric in the Prometheus format, it is also possible to export metric data programmatically in your application code.

## Client libraries
The Prometheus project provides client libraries which are either official or maintained by third-parties. There are libraries for major languages like Java, Go, Python, PHP, and .NET/C#.

Even if you don’t plan to provide your own metrics, those libraries already export some basic metrics based on the language. For Go , default metrics about memory management (heap, garbage collection) and thread pools can be collected. The same applies to Java .

## Specifications and conventions
Application metrics or metrics in general can contain confidential information, therefore endpoints should be protected from unauthenticated users. This can be achieved either by exposing the metrics on a different port, which is only reachable by prometheus or by protecting the metrics endpoints with some sort of authentication.

There are some guidelines and best practices how to name your own metrics. Of course, the specifications of the datamodel must be followed and applying the best practices about naming is not a bad idea. All those guidelines and best practices are now officially specified in openmetrics.io .

Following these principles is not (yet) a must, but it helps to understand and interpret your metrics.

## Best practices
Though implementing a metric is an easy task from a technical point of view, it is not so easy to define what and how to measure. If you follow your existing log statements and if you define an error counter to count all errors and exceptions , then you already have a good base to see the internal state of your application.

### The four golden signals
Another approach to define metrics is based on the four golden signals :

> - Latency
> - Traffic
> - Errors
> - Saturation

There are other methods like RED or USE that go into the same direction.

***

# 5.1 Tasks: Instrumenting

## Task 5.1.1: Clone and test the Go example app

Change to the downloads directory and clone the empty Go example git repository

`cd ~/downloads`

`git clone https://github.com/acend/prometheus-training-go-instrumentation.git`


Change into the freshly cloned git repository

`cd prometheus-training-go-instrumentation`


This application creates a simple webserver which listens at http://localhost:8083 and returns the string Prometheus Training after a random interval between 0 and 1000 milliseconds. You can test the app by running it using go run main.go and then (in a separate terminal) issue a request using the command curl http://localhost:8083.

## Task 5.1.2: Instrument the Go example app

The Prometheus Go client library provides an easy way to add prometheus metrics to our own applications. In this task we will extend a simple Go application expose these metrics.

Task description:

> - Import the Prometheus Go client library in our example app
> - Add a http handler for the /metrics endpoint to output the default metrics provided by the promhttp.Handler() handler
> - Check if the app returns metrics at http://localhost:8083/metrics

In order to collect the metrics and use the promhttp.Handler function we first need to download the prometheus/promhttp library:

`go get github.com/prometheus/client_golang/prometheus/promhttp`


Next we replace the line // insert prometheus library here in the main.go file with the following line:

`    "github.com/prometheus/client_golang/prometheus/promhttp"`


Finally we need to add a handler for the /metrics endpoint. Replace the line // insert additional handlers here in the main.go file with the following line:

`    http.Handle("/metrics", promhttp.Handler())`


The main.go of our example app should now look like this:

    >
    package main

    import (
        "fmt"
        "net/http"
        "math/rand"
        "time"
        "github.com/prometheus/client_golang/prometheus/promhttp"
    )

    func main() {
        http.HandleFunc("/", ServeHandler)
        http.Handle("/metrics", promhttp.Handler())
        err := http.ListenAndServe(":8083", nil)
        if err != nil {
            fmt.Println(err)
        }
    }

    func ServeHandler(w http.ResponseWriter, r *http.Request) {
        rand.Seed(time.Now().UnixNano())
        n := rand.Intn(1000)
        time.Sleep(time.Duration(n)*time.Millisecond)
        fmt.Fprintf(w, "Prometheus Training")
    }


Now we should be able to run our application:

`go run main.go`


Using a second terminal window we can now query the /metrics endpoint of our app and should receive a number of Prometheus metrics

`curl localhost:8083/metrics`


Expected result:

>
    curl localhost:8083/metrics
    # HELP go_gc_cycles_automatic_gc_cycles_total Count of completed GC cycles generated by the Go runtime.
    # TYPE go_gc_cycles_automatic_gc_cycles_total counter
    go_gc_cycles_automatic_gc_cycles_total 0
    # HELP go_gc_cycles_forced_gc_cycles_total Count of completed GC cycles forced by the application.
    # TYPE go_gc_cycles_forced_gc_cycles_total counter
    go_gc_cycles_forced_gc_cycles_total 0
    # HELP go_gc_cycles_total_gc_cycles_total Count of all completed GC cycles.
    # TYPE go_gc_cycles_total_gc_cycles_total counter
    go_gc_cycles_total_gc_cycles_total 0
    # HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
    # TYPE go_gc_duration_seconds summary
    go_gc_duration_seconds{quantile="0"} 0
    go_gc_duration_seconds{quantile="0.25"} 0
    go_gc_duration_seconds{quantile="0.5"} 0
    go_gc_duration_seconds{quantile="0.75"} 0
    go_gc_duration_seconds{quantile="1"} 0
    go_gc_duration_seconds_sum 0
    go_gc_duration_seconds_count 0
    ...


## Task 5.1.3: Scrape config

Task description:

> - Configure Prometheus to scrape the metrics endpoint of the Go Application
> - Add a new Job to the scrape_configs in your /etc/prometheus/prometheus.yml:

>
    scrape_configs:
    ...
    - job_name: "go_app"
        metrics_path: "/metrics"
        static_configs:
        - targets:
            - "localhost:8083"
    ...

Note that you can always validate the configuration with:

`promtool check config /etc/prometheus/prometheus.yml`


Do not forget to reload or restart the Prometheus server.

## Task 5.1.4: Metric names

Study the following metrics and decide if the metric name is ok

`http_requests{handler="/", status="200"}`

`http_request_200_count{handler="/"}`

`go_memstats_heap_inuse_megabytes{instance="localhost:9090",job="prometheus"}`

`prometheus_build_info{branch="HEAD",goversion="go1.19.1",instance="localhost:9090",job="prometheus",revision="6d7f26c46ff70286944991f95d791dff03174eea",version="2.39.0"}`

`prometheus_config_last_reload_success_timestamp{instance="localhost:9090",job="prometheus"}`

`prometheus_tsdb_lowest_timestamp_minutes{instance="localhost:9090",job="prometheus"}`


> - The _total suffix should be appended, so http_requests_total{handler="/", status="200"} is better.
> - There are two issues in http_request_200_count{handler="/"}: The _count suffix is foreseen for histograms, counters can be suffixed with _total. Second, status information should not be part of the metric name, a label {status="200"} is the better option.
> - The base unit is bytes not megabytes, so go_memstats_heap_inuse_bytes is correct.
> - Everything is ok with prometheus_build_info and its labels. It’s a good practice to export such base information with a gauge.
> - In prometheus_config_last_reload_success_timestamp, the base unit is missing, correct is prometheus_config_last_reload_success_timestamp_seconds.
> - The base unit is seconds for timestamps, so prometheus_tsdb_lowest_timestamp_seconds is correct.

## Task 5.1.5: Metric names (optional)

What kind of risk do you have, when you see such a metric

`http_requests_total{path="/etc/passwd", status="404"} 1`


There is no potential security vulnerability from exposing the /etc/passwd path, which seems to be handled appropriately in this case: no password is revealed.

From a Prometheus point of view, however, there is the risk of a DDoS attack: An attacker could easily make requests to paths which obviously don’t exist. As every request and therefore path is registered with a label, many new time series are created which could lead to a cardinality explosion and finally to out-of-memory errors.

It’s hard to recover from that!

For this case, it’s better just to count the 404 requests and to lookup the paths in the log files.

`http_requests_total{status="404"} 15`


## Task 5.1.6: Custom metric (optional)

Task description:

> - Create a custom histogram metric with the name training_app_request_duration_seconds which measures the time (per request) used by the ServeHandler handler
> - Make a couple of requests to the example application
> - Calculate the 85th percentile of the request duration of the example app

Download the following two prometheus libraries and add them to the imports in the main.go file:

> - github.com/prometheus/client_golang/prometheus/promauto
> - github.com/prometheus/client_golang/prometheus

`go get github.com/prometheus/client_golang/prometheus/promauto \`

`  github.com/prometheus/client_golang/prometheus`


Add the imports to main.go (only the relevant section is shown in the example below)

>
    ...
    import (
        ...
        "github.com/prometheus/client_golang/prometheus/promauto"
        "github.com/prometheus/client_golang/prometheus"
    ...


Add a new variable for the histogram metric to the main.go file (only the relevant section is shown in the example below):

>
    ...
    var (
        responseLatency = promauto.NewHistogram(prometheus.HistogramOpts{
            Name: "training_app_request_duration_seconds",
            Help: "example app response latencies",
        })

    )

    func main() {
    ...


Extend the existing ServeHandler function to measure the time between the start end the end of the function and update the histogram accordingly: main.go (only the relevant section is shown in the example below)

>
    ...
    func ServeHandler(w http.ResponseWriter, r *http.Request) {
        requestReceived := time.Now()
        rand.Seed(time.Now().UnixNano())
        n := rand.Intn(1000)
        time.Sleep(time.Duration(n)*time.Millisecond)
        fmt.Fprintf(w, "Prometheus Training")
        responseLatency.Observe(time.Since(requestReceived).Seconds())
    }


By now your main.go file should look like this:

>
    package main

    import (
        "fmt"
        "net/http"
        "math/rand"
        "time"
        "github.com/prometheus/client_golang/prometheus/promhttp"
        "github.com/prometheus/client_golang/prometheus/promauto"
        "github.com/prometheus/client_golang/prometheus"
    )

    var (
        responseLatency = promauto.NewHistogram(prometheus.HistogramOpts{
            Name: "training_app_request_duration_seconds",
            Help: "example app response latencies",
        })

    )

    func main() {
        http.HandleFunc("/", ServeHandler)
        http.Handle("/metrics", promhttp.Handler())
        err := http.ListenAndServe(":8083", nil)
        if err != nil {
            fmt.Println(err)
        }
    }

    func ServeHandler(w http.ResponseWriter, r *http.Request) {
        requestReceived := time.Now()
        rand.Seed(time.Now().UnixNano())
        n := rand.Intn(1000)
        time.Sleep(time.Duration(n)*time.Millisecond)
        fmt.Fprintf(w, "Prometheus Training")
        responseLatency.Observe(time.Since(requestReceived).Seconds())
    }


Next we have to build and run the app again:

`go run main.go`


Now let’s create a couple of requests to our new endpoint. Make sure to run those commands from a second terminal window, while the Go application is still running.

`for i in {1..100}; do (curl http://localhost:8083/ -o /dev/null -s &) done`


Verify the Prometheus metrics endpoint and look for a metric with the name training_app_request_duration_seconds_bucket

`curl http://localhost:8083/metrics`


Expected result:

>
    ...
    # HELP training_app_request_duration_seconds example app response latencies
    # TYPE training_app_request_duration_seconds histogram
    training_app_request_duration_seconds_bucket{le="0.005"} 0
    training_app_request_duration_seconds_bucket{le="0.01"} 1
    training_app_request_duration_seconds_bucket{le="0.025"} 1
    training_app_request_duration_seconds_bucket{le="0.05"} 4
    training_app_request_duration_seconds_bucket{le="0.1"} 10
    training_app_request_duration_seconds_bucket{le="0.25"} 24
    training_app_request_duration_seconds_bucket{le="0.5"} 55
    training_app_request_duration_seconds_bucket{le="1"} 100
    training_app_request_duration_seconds_bucket{le="2.5"} 100
    training_app_request_duration_seconds_bucket{le="5"} 100
    training_app_request_duration_seconds_bucket{le="10"} 100
    training_app_request_duration_seconds_bucket{le="+Inf"} 100
    training_app_request_duration_seconds_sum 48.73099980000001
    training_app_request_duration_seconds_count 100
    ...


Let us now calculate the 85th percentile by navigating to the Prometheus Web Console by entering the following query in the Prometheus expression browser:

`histogram_quantile(0.85, training_app_request_duration_seconds_bucket)`


The result describes the number of seconds it took to process 85% of all requests (15% of the requests were slower than the calculated result).

***

# 6. Visualization

Our goal with this lab is to give you a brief overview how to visualize your Prometheus time series produced in the previous labs. For a more detailed tutorial, please refer to the Grafana tutorials .

Grafana is already installed and running on your machine. Login to your Grafana instance on http://localhost:3000/ . Use your personal credentials to access the Grafana login page. Then use the Grafana default credentials (username: admin, password: admin) to log in to Grafana.

Note: Wou will have to authenticate twice to access Grafana. First use your personal credentials (same used in the earlier labs to log in to Prometheus or Alertmanager) then on the Grafana login page use the Grafana default credentials (admin/admin)

## Useful links and guides

> - Prometheus data source
> - Grafana dashboards
> - Grafana provisioning

***

# 6.1 Tasks: Grafana intro

## Task 6.1.1 Configure Prometheus to scrape Grafana metrics

This is repetition. Grafana instruments the Prometheus client library and provides a variety of metrics at the /metrics endpoint: http://localhost:3000/metrics

Configure Prometheus to scrape these metrics

Add the following static_config to your /etc/prometheus/prometheus.yml

>
    ...
    - job_name: 'grafana'
        static_configs:
        - targets: ['localhost:3000']
    ...


Reload Prometheus

`sudo systemctl reload prometheus`


Check if the Grafana instance appears in the targets section of Prometheus (http://localhost:9090/targets ). In addition you can use the following query to show list all metrics of the new target:

`{instance= "localhost:3000"}`

## Task 6.1.2 Prometheus datasource

To be able to use our Prometheus Server as datasource in Grafana dashboards you have to configure it in Grafana.

Add your Prometheus server as a data source to Grafana in the web UI.

Open the Grafana Web UI and navigate to Configuration (Icon on the left navigation menu that looks like a gear) > Data sources > Add data source

> - Choose Prometheus as data source type
> - Set URL to http://localhost:9090
> - Hit Save & test

Visualize the Prometheus metric :node_memory_MemAvailable_bytes:sum under Explore to test your data source.

Navigate to Explore (Icon on the left navigation menu that looks like a compass)

> - Choose Prometheus as data source
> - Select the metric :node_memory_MemAvailable_bytes:sum in the Select metric dropdown
> - Hit Run query

## Task 6.1.3 Import a dashboard

Task description:

> - Import a dashboard from https://grafana.com/grafana/dashboards to your Grafana instance
> - Display the metrics in the imported dashboard between 5 minutes and 10 minutes ago

Note: You can import the Prometheus Internal Stats dashboard, which will present you useful metrics about your Prometheus server

> - Navigate to https://grafana.com/grafana/dashboards/11449 and copy the dashboard ID
> - On your Grafana web UI
>> - Navigate to + (left navigation menu) > Import
>> - Add the copied ID to the Import via grafana.com field
>> - Hit Load
> - Choose your Prometheus data source and hit Import
> - Open the dashboard time control (to the upper right)
>> - Set From to now-10m
>> - Set To to now-5m
>> - Hit Apply time range

***

# 6.2 Tasks: Grafana dashboards

## Task 6.2.1 Create your first dashboard

In this task you’re going to create your first own dashboard happy_little_dashboard. You will add the panel CPU Utilisation with the metric instance:node_cpu_utilisation:rate5m.

Navigate to Dashboards (Icon with the four squares on the left navigation menu)> New Dashboard:

> - Select Add a new panel
> - Add the rule instance:node_cpu_utilisation:rate5m in the Select metric dropdown
> - Set the panel title to CPU Utilisation under Panel options > Title (you may need to open the options pane with the < button on the right hand side just below the Apply button)
> - Save the dashboard and give it the name happy_little_dashboard

## Task 6.2.2 Add a Gauge panel to the dashboard

Task description:

Add another panel to the existing happy_little_dashboard with the panel name Memory Available. Display the metric :node_memory_MemAvailable_bytes:sum and change the panel type to Gauge and display it in %. Define the following thresholds:
>
    0% (red)
    20% (orange)
    50% (green)

> - Hit Add panel (top navigation menu) > Add a new panel
>> - Add the rule :node_memory_MemAvailable_bytes:sum to the Metrics browser field
>> - Set the panel title to Memory Available under Panel options > Title (you may need to open the options pane with the < button on the right hand side just below the Apply button)
>> - Define unit under Standard options > Unit > Misc / Percent (0.0-1.0)
>> - Choose Gauge in the dropdown menu just below the Apply button

> - Add 0.2 and 0.5 thresholds under Thresholds
>> - Choose Red for Base
>> - Choose Orange for 0.2
>> - Choose Green for 0.5

> - Save the dashboard

***

# 7. Prometheus in Kubernetes

## Prometheus in Kubernetes

We will use minikube to start a minimal Kubernetes environment. If you are a novice in Kubernetes, you may want to use the kubectl cheatsheet

### Minikube

Minikube is already started and configured. When you restart your virtual machine, you might need to start it manually.

`minikube start \`

`--kubernetes-version=v1.23.1 \`

`--memory=6g \`

`--cpus=4 \`

`--bootstrapper=kubeadm \`

`--extra-config=kubelet.authentication-token-webhook=true \`

`--extra-config=kubelet.authorization-mode=Webhook \`

`--extra-config=scheduler.address=0.0.0.0 \`

`--extra-config=controller-manager.address=0.0.0.0`


Check if you can connect to the API and verify the minikube master node is in ready state.

`kubectl get nodes`

>
    NAME       STATUS   ROLES                  AGE    VERSION
    minikube   Ready    control-plane,master   2m2s   v1.23.1


Deploy the Prometheus operator stack, consisting of:

> - Prometheus Operator deployment
> - Prometheus Operator ClusterRole and ClusterRoleBinding

> - CustomResourceDefinitions
>> - Prometheus : Manage the Prometheus instances
>> - Alertmanager : Manage the Alertmanager instances
>> - ServiceMonitor : Generate Kubernetes service discovery scrape configuration based on Kubernetes service definitions
>> - PrometheusRule : Manage the Prometheus rules of your Prometheus
>> - AlertmanagerConfig : Add additional receivers and routes to your existing Alertmanager configuration
>> - PodMonitor : Generate Kubernetes service discovery scrape configuration based on Kubernetes pod definitions
>> - Probe : Custom resource to manage Prometheus blackbox exporter targets
>> - ThanosRuler : Manage Thanos rulers

`git clone https://github.com/prometheus-operator/kube-prometheus.git ~/work/kube-prometheus`

`cd ~/work/kube-prometheus`

`git checkout release-0.10`

`kubectl create -f manifests/setup`


The manifest will deploy a complete monitoring stack consisting of:

> - Two Prometheis - What is the plural of Prometheus? ;)
> - Alertmanager cluster
> - Grafana
> - kube-state metrics exporter
> - node_exporter
> - blackbox exporter
> - A set of default PrometheusRules
> - A set of default dashboards

`kubectl create -f manifests/`


By default, Prometheus is only allowed to monitor the default, monitoring and kube-system namespaces. Therefore we will add the necessary ClusterRoleBinding to grant Prometheus access to cluster-wide resources. Also we will create the needed ingress definitions for you, which will expose the monitoring components.

`kubectl -n monitoring apply -f \`

`https://raw.githubusercontent.com/puzzle/prometheus-training/main/content/en/docs/07/resources.yaml`


Wait until all pods are running

`watch kubectl -n monitoring get pods`


Check if you can access the Prometheus web interface at http://localhost:19090

Check access to Alertmanager at http://localhost:19093

Check access to Grafana at http://localhost:13000

Note: Use the default Grafana logging credentials and change the password

> - username: admin
> - password: admin

***

# 7.1 Tasks: Prometheus Operator basics

## Task 7.1.1: Display pod metrics in Kubernetes Grafana

The Prometheus operator stack provides a few generic dashboards for your Kubernetes cluster deployment. These dashboards provide you with information about the resource usage of Kubernetes infrastructure components or your deployed apps. They also show you latency and availability of Kubernetes core components.

Task description:

> - Navigate to your Kubernetes Grafana
> - Find a dashboard that displays compute resources per namespace and pod
> - Take a look at the metrics from the monitoring namespace

Note: The initial password is admin. You need to change it after the first login.

> - Use the search function (magnifying glass) on the left side and hit Search
> - The dashboard name is Kubernetes / Compute Resources / Namespace (Pods) and can be found in the Default directory
> - Select monitoring in the namespace drop-down

You get usage metrics for CPU and memory as well as network statistics per pod in the namespace Monitoring.

## Task 7.1.2: Configure Alertmanager and Prometheus storage

By default, the Prometheus operator stack does not persist the data of the deployed monitoring stack. Therefore, any pod restart would result in a reset of all data. Let’s configure persistence for Prometheus and Alertmanager.

Task description:

> - See this example of how to configure storage for Prometheus
> - Set the Prometheus volume size to 20Gi
> - Set the Alertmanager volume size to 1Gi

Note: To get the custom resources name of your Alertmanager or Prometheus run:

`kubectl -n monitoring get prometheuses`

`kubectl -n monitoring get alertmanagers`


The default text editor in Kubernetes is Vim. If you are not familiar with Vim, you can switch to Nano or Emacs by setting the KUBE_EDITOR environment variable. Example to use nano:

`echo 'export KUBE_EDITOR="nano"' >> ~/.bashrc`

`source ~/.bashrc`


Alternatively, you can export your resources, edit them in Theia, and apply them to the Kubernetes cluster. For example:

`kubectl get deployment <name> -o yaml > ~/work/deployment.yaml`

`kubectl apply -f ~/work/deployment.yaml`


Custom resources can be changed by using kubectl edit.

`kubectl -n monitoring edit <type> <name>`


If you want to edit the Prometheus custom resource you would use

`kubectl -n monitoring edit prometheuses k8s`


Define the storage size for your Prometheis

`kubectl -n monitoring edit prometheuses k8s`

>
    ...
    spec:
    ...
    storage:
        volumeClaimTemplate:
        spec:
            resources:
            requests:
                storage: 20Gi
    ...
    Copy
    Define the storage size for your Alertmangers

    kubectl -n monitoring edit alertmanagers main
    Copy
    ...
    spec:
    ...
    storage:
        volumeClaimTemplate:
        spec:
            resources:
            requests:
                storage: 1Gi
    ...


Make sure Kubernetes provisioned the Persistent Volumes

`kubectl -n monitoring get pvc`


Check if the volume is available inside the pod with running df -h /prometheus inside the first Prometheus pod.

`kubectl -n monitoring exec prometheus-k8s-0 -c prometheus -- df -h /prometheus`

>
    Filesystem                Size      Used Available Use% Mounted on
    /dev/sda1                20.0G      8.4G      1.6G  84% /prometheus


Note: We use Minkube, which for demonstration purposes uses the /dev/sda1 disk of your virtual machine as the storage backend for all Persistent Volumes. Therefore, you will always see the size of /dev/sda1 when checking from inside a container. If you use an appropriate storage backend, the size inside the container will correspond to the size of your Persistent Volume.

## Task 7.1.3: Configure Prometheus Retention

By default, the Prometheus operator stack will set the retention of your metrics to 24h. As we have now 20Gi of disk space available, we can increase the retention. Read about retention operational-aspects for options to manage retention.

Task description:

> - Set Prometheus retention time to two days and retention size to 9Gi

Note: Check documentation for available options
Set the following options

`kubectl -n monitoring edit prometheus k8s`

>
    ...
    spec:
    ...
    retention: 2d
    retentionSize: 9GB
    ...

Verify that the pods are redeployed with kubectl -n monitoring get pods and that the retention parameter is set in the newly created pods.

`kubectl -n monitoring describe pods prometheus-k8s-0`


The output should contain the following lines:

>
    ...
    Containers:
    ...
    prometheus:
        ...
        Args:
        ...
        --storage.tsdb.retention.size=9GB
        --storage.tsdb.retention.time=2d
        ...


## Task 7.1.4: Configure additional Alertmanager receiver

We can manage the Kubernetes Alertmanager via several approaches. In this task, we will learn how to add an additional receiver using alertmanagerConfig custom resource . First we need do define the alertmanagerConfigSelector label in the Alertmanager. This must match the labels defined in our alertmanagerConfig.

`kubectl -n monitoring edit alertmanagers main`


    ...
    spec:
    ...
    alertmanagerConfigSelector:
        matchLabels:
        alertmanagerConfig: training
    ...


Task description:

> - Configure Alertmanger to send all alerts from the monitoring namespace to MailCatcher
> - Create a AlertmanagerConfig custom resource. See example as reference
> - Name the resource mailcatcher
> - Define the following route and receiver

>
    ...
    route:
        receiver: 'mailcatcher'
    receivers:
        - name: 'mailcatcher'
        emailConfigs:
            - to: 'alert@localhost'
            from: 'prometheus-operator@localhost'
            smarthost: '192.168.49.1:1025'
            requireTLS: false

Note: When you create an AlertmanagerConfig, it will only match alerts that have the namespace label set to the scope in which the AlertmanagerConfig is defined. In our case:

>
    ...
    route:
        ...
        routes:
        - receiver: monitoring-mailcatcher-mailcatcher
        match:
            namespace: monitoring
        ...


Add the AlertmanagerConfig

`curl -o ~/work/mailcatcher.yml \`

`https://raw.githubusercontent.com/puzzle/prometheus-training/main/content/en/docs/07/labs/mailcatcher.yml`

`kubectl -n monitoring create -f ~/work/mailcatcher.yml`


mailcatcher.yml content:

>
    ---
    apiVersion: monitoring.coreos.com/v1alpha1
    kind: AlertmanagerConfig
    metadata:
    name: mailcatcher
    labels:
        alertmanagerConfig: training
    spec:
    route:
        receiver: mailcatcher
    receivers:
        - name: mailcatcher
        emailConfigs:
            - to: alert@localhost
            from: prometheus-operator@localhost
            smarthost: 192.168.49.1:1025
            requireTLS: false

*Optional: You can add an alert to check your configuration using the amtool and check if the MailCatcher received the mail. It can take up to 5 minutes as the alarms are grouped together based on the group_interval . E.g.*

`kubectl -n monitoring exec alertmanager-main-0  -c alertmanager -- \`

`amtool alert add --alertmanager.url=http://localhost:9093 alertname=test namespace=monitoring severity=critical`


## Task 7.1.5: Check if Alertmanager is running clustered (optional)

The Prometheus operator stack deployed three Alertmanagers. Normally there would be further configuration needed to make sure that these instances running clustered. But as we are running Alertmanager managed by Prometheus operator this should be done automatically.

Task description: Investigate if Alertmanger is clustered and which paramaters have been set by the operator

The Alertmanager custom resource has 3 replicas configured

`kubectl -n monitoring get alertmanager main -o yaml`

>
    ...
    spec:
    ...
    replicas: 3
    ...


The operator makes sure that the Alertmanagers know about each other and generates the necessary configuration to form a cluster. Let’s review the pod definition:

`kubectl -n monitoring get pods alertmanager-main-0 -o yaml`

>
    ...
    spec:
    ...
    containers:
    - args:
        ...
        - --cluster.listen-address=[$(POD_IP)]:9094
        - --cluster.peer=alertmanager-main-0.alertmanager-operated:9094
        - --cluster.peer=alertmanager-main-1.alertmanager-operated:9094
        - --cluster.peer=alertmanager-main-2.alertmanager-operated:9094
    ...

***

# 7.2 Tasks: kube-prometheus setup
The kube-prometheus stack already provides an extensive Prometheus setup and contains a set of default alerts and dashboards from Prometheus Monitoring Mixin for Kubernetes . The following targets will be available.

kube-state-metrics: Exposes metadata information about Kubernetes resources. Used, for example, to check if resources have the expected state (deployment rollouts, pods CrashLooping) or if jobs fail.

>
    # Example metrics
    kube_deployment_created
    kube_deployment_spec_replicas
    kube_daemonset_status_number_misscheduled
    ...

kubelet/cAdvisor: Advisor exposes usage and performance metrics about running container. Commonly used to observe memory usage or CPU throttling .

>
    # Example metrics
    container_cpu_cfs_throttled_periods_total
    container_memory_working_set_bytes
    container_fs_inodes_free
    ...

kubelet: Exposes general kubelet related metrics. Used to observe if the kubelet and the container engine is healthy.

>
    # Example metrics
    kubelet_docker_operations_duration_seconds_bucket
    kubelet_runtime_operations_total
    ...

apiserver: Metrics from the Kubernetes API server. Commonly used to catch errors on resources or problems with latency.

>
    # Example metrics
    apiserver_request_duration_seconds_bucket
    apiserver_request_total{code="200",...}
    ...


kubelet/probes: Expose metrics about Kubernetes liveness, readiness and startup probes Normally you would not alert on Kubernetes probe metrics, but on container restarts exposed by kube-state-metrics.

>
    # Example metrics
    prober_probe_total{probe_type="Liveness", result="successful",...}
    prober_probe_total{probe_type="Startup", result="successful",...}
    ...


blackbox-exporter: Exposes default metrics from blackbox-exporter. Can be customized using the Probe custom resource.

>
    # Example metrics
    probe_http_status_code
    probe_http_duration_seconds
    ...


node-exporter: Exposes the hardware and OS metrics from the nodes running Kubernetes.

>
    # Example metrics
    node_filesystem_avail_bytes
    node_disk_io_now
    ...

alertmanager-main/grafana/prometheus-k8s/prometheus-operator/prometheus-adapter: Exposes all monitoring stack component metrics.

>
    # Example metrics
    alertmanager_alerts_received_total
    alertmanager_config_last_reload_successful
    ...
    grafana_alerting_active_alerts
    grafana_datasource_request_total
    ...
    prometheus_config_last_reload_successful
    prometheus_rule_evaluation_duration_seconds
    ...
    prometheus_operator_reconcile_operations_total
    prometheus_operator_managed_resources
    ...

## Task 7.2.1: Container restart alerting rule

Navigate to the Prometheus user interface and take a look at the provided default Prometheus rules.

Note: Search for an alert with CrashLooping in its name
Task description:

> - Investigate if there is a default Alerting rule configured to monitor container restarts
> -Which exporter exposes the required metrics?
The Alerting rule is called KubePodCrashLooping and the PromQL defined for the rule looks as follows:

`max_over_time(kube_pod_container_status_waiting_reason{job="kube-state-metrics",reason="CrashLoopBackOff"}[5m]) >= 1`

If you take a look at the query, you will see that there is a filter that only uses metrics from the kube-state-metrics exporter.

## Task 7.2.2: Memory usage of Prometheus

Note: Search for a metric with memory_working_set in its name
Task description:

> -Display the memory usage of both Prometheus pods
> -Use a filter to just display metrics from the prometheus containers

`container_memory_working_set_bytes{pod=~"prometheus-k8s-.*", container="prometheus"}`


Your query returns two time series per Prometheus replica with the same value but different labels. One has additional information and allows you to filter the Docker ID.

>
    # Docker container ID as an additional information
    id="/docker/34750eee3d37c8cd3594fc46bb20ed166374ece7737c590b81ed1847aaa21d50/kubepods/burstable/pode2ab8dc5-ad8e-4b5c-9b0f-49c3bb5a8a34/dce2373557fb05e0bc9819b9f786f6e3bcc882280ee2eb9267edb7040886a55b"
    # Kubernetes pod information
    id="/kubepods/burstable/pode2ab8dc5-ad8e-4b5c-9b0f-49c3bb5a8a34/dce2373557fb05e0bc9819b9f786f6e3bcc882280ee2eb9267edb7040886a55b"


## Task 7.2.3: Kubernetes pod count

Task description:

> -Display how many pods are currently running on your Kubernetes platform

There are different ways to archive this. You can for example query all running containers and group them by pod and namespace.

`count(sum(kube_pod_container_status_running == 1) by (pod,namespace))`


You may also sum() all running pods on your Kubernetes nodes

`sum(kubelet_running_pods)`


## Task 7.2.4: Add a custom Prometheus rule

The Prometheus operator allows you to extend the existing Prometheus rules with your own rules using the PrometheusRule custom resource . You can take a look at existing PrometheusRules or at this example about the format of PrometheusRules.

Task description:

> -Add a custom Prometheus rule to the monitoring stack, which checks if you have reached the defined retention size

Note: You can use the prometheus_tsdb_size_retentions_total metric

> -Set the labels prometheus: k8s and role: alert-rules on your PrometheusRule to match the resource with your Prometheus configuration

See Prometheus custom resource as reference

`kubectl -n monitoring edit prometheuses k8s`

>
    ...
    spec:
    ...
    ruleSelector:
        matchLabels:
        prometheus: k8s
        role: alert-rules
    ...


There are different ways to archive this. The first approach is to just check the number of times that blocks were deleted because the maximum number of bytes was exceeded.

`curl -o ~/work/prometheus-custom-rule.yml \`

`https://raw.githubusercontent.com/puzzle/prometheus-training/main/content/en/docs/07/labs/prometheus-custom-rule.yml`

`kubectl -n monitoring create -f ~/work/prometheus-custom-rule.yml`


prometheus-custom-rule.yml content:

>
    ---
    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
    name: custom-rule
    namespace: monitoring
    labels:
        prometheus: k8s
        role: alert-rules
    spec:
    groups:
        - name: custom.rules
        rules:
            - alert: PrometheusReachedRetentionSize
            annotations:
                summary: Prometheus deleted blocks because the maximum number of bytes
                was exceeded in {{ $labels.namespace }} {{ $labels.pod }}
            expr: rate(prometheus_tsdb_size_retentions_total[5m]) > 0
            labels:
                severity: warning


Another approach is to alert, when the on-disk time series database size is greater than prometheus_tsdb_retention_limit_bytes. For example:

`prometheus_tsdb_storage_blocks_bytes >= prometheus_tsdb_retention_limit_bytes`


In the Prometheus web UI there is an Alerts menu item which shows you information about the configured alert.

***

# 8. Prometheus in Kubernetes App Monitoring

## Collecting Application Metrics

When running applications in production, a fast feedback loop is a key factor. The following reasons show why it’s essential to gather and combine all sorts of metrics when running an applications in production:

> - To make sure that an application runs smoothly
> - To be able to see production issues and send alerts
> - To debug an application
> - To take business and architectural decisions
> - Metrics can also help to decide when to scale applications

As we saw in Lab 5 - Instrumenting with client libraries Application Metrics (e.g. Request Count on a specific URL, GC metrics, or even Custom Metrics and many more) are collected within the application. There are a lot of frameworks and client libraries available, which integrate nicely into different application stacks.

The instrumented application provides Prometheus scrapable application metrics.

Create a namespace where the example application can be deployed to.

`kubectl create namespace application-metrics`


Deploy the Acend example Python application, which provides application metrics at /metrics:

`kubectl -n application-metrics create deployment example-web-python \`

`--image=quay.io/acend/example-web-python`


Use the following command to verify the deployment, that the pod example-web-python is Ready and Running. (use CTRL C to exit the command)

`kubectl -n application-metrics get pod -w`


We also need to create a Service for the new application. Create a file with the name ~/work/service.yaml with the following content:

>
    ---
    apiVersion: v1
    kind: Service
    metadata:
    labels:
        app: example-web-python
        prometheus-monitoring: 'true'
    name: example-web-python
    spec:
    ports:
        - name: http
        port: 5000
        protocol: TCP
        targetPort: 5000
    selector:
        app: example-web-python
    type: NodePort


Create the Service with the following command:

`kubectl apply -f ~/work/service.yaml -n application-metrics`


This created a so-called Kubernetes Service

`kubectl -n application-metrics get services`


Which gives you an output similar to this:

>
    NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    example-web-python   NodePort   10.101.249.125   <none>        5000:31626/TCP   2m9s


Our example application can now be reached on the port 31626. This may be different in your setup.

We can now get the exposed url with the minikube service command:

`minikube service example-web-python --url -n application-metrics`

`# http://192.168.49.2:31626`


Use curl and verify the successful deployment of our example application:

`curl $(minikube service example-web-python --url -n application-metrics)/metrics`


Should result in something like:
>
    # HELP python_gc_objects_collected_total Objects collected during gc
    # TYPE python_gc_objects_collected_total counter
    python_gc_objects_collected_total{generation="0"} 541.0
    python_gc_objects_collected_total{generation="1"} 344.0
    python_gc_objects_collected_total{generation="2"} 15.0
    ...


Since our newly deployed application now exposes metrics, the next thing we need to do, is to tell our Prometheus server to scrape metrics from the Kubernetes deployment. In a highly dynamic environment like Kubernetes this is done with so called Service Discovery.

## Service Discovery
When configuring Prometheus to scrape metrics from Containers deployed in a Kubernetes Cluster it doesn’t really make sense to configure every single target manually. That would be far too static and wouldn’t really work in a highly dynamic environment. Instead it makes sense to use a similar concept, like we used in Lab 1 - Dynamic configuration .

In fact, we tightly integrate Prometheus with Kubernetes and let Prometheus discover the targets, which need to be scraped automatically via the Kubernetes API.

The tight integration between Prometheus and Kubernetes can be configured with the Kubernetes Service Discovery Config .

Now we instruct Prometheus to scrape our application metrics from the sample application by creating a ServiceMonitor.

ServiceMonitors are Kubernetes custom resources, which basically represent the scrape_config and look like this:

>
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
    labels:
        app.kubernetes.io/name: example-web-python
    name: example-web-python-monitor
    spec:
    endpoints:
    - interval: 30s
        port: http
        scheme: http
        path: /metrics
    selector:
        matchLabels:
        prometheus-monitoring: 'true'


## How does it work

The Prometheus Operator watches namespaces for ServiceMonitor custom resources. It then updates the Service Discovery configuration of the Prometheus server(s) accordingly.

The selector part in the ServiceMonitor defines which Kubernetes Services will be scraped.

>
    # servicemonitor.yaml
    ...
    selector:
        matchLabels:
        prometheus-monitoring: 'true'
    ...


And the corresponding Service

>
    apiVersion: v1
    kind: Service
    metadata:
    name: example-web-python
    labels:
        prometheus-monitoring: 'true'
    ...


This means Prometheus scrapes all Endpoints where the prometheus-monitoring: 'true' label is set.

The spec section in the ServiceMonitor resource allows to further configure the targets. In our case Prometheus will scrape:

> - Every 30 seconds
> - Look for a port with the name http (this must match the name in the Service resource)
> - Scrape metrics from the path /metrics using http

## Best practices

Use the common k8s labels https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/

If possible, reduce the number of different ServiceMonitors for an application and thereby reduce the overall complexity.

> - Use the same matchLabels on different Services for your application (e.g. Frontend Service, Backend Service, Database Service)
> - Also make sure the ports of different Services have the same name
> - Expose your metrics under the same path

Avoid relabeling and use standards or change the metric labels within the exporter.

***

# 8.1 Tasks: Application Monitoring

## Task 8.1.1: Create a ServiceMonitor

Task description:

Create a ServiceMonitor for the example application

> - Create a ServiceMonitor, which will configure Prometheus to scrape metrics from the example-web-python application every 30 seconds.

*hint: kubectl -n application-metrics apply -f my_file.yaml will create a resource in the Kubernetes namespace*

For this to work, you need to ensure:

> - The example-web-python Service is labeled correctly and matches the labels you’ve defined in your ServiceMonitor.
> - The port name in your ServiceMonitor configuration matches the port name in the Service definition.

*hint: check with kubectl get service example-web-python -n application-metrics -o yaml*

> - Verify the target in the Prometheus user interface

Create the following ServiceMonitor (~/work/servicemonitor.yaml) in the application-metrics namespace

>
    ---
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
    labels:
        app.kubernetes.io/name: example-web-python
    name: example-web-python-monitor
    spec:
    endpoints:
        - interval: 30s
        port: http
        scheme: http
        path: /metrics
    selector:
        matchLabels:
        prometheus-monitoring: 'true'


Apply it using the following command:

`kubectl -n application-metrics apply -f ~/work/servicemonitor.yaml`


Verify that the target gets scraped in the Prometheus user interface . Target name: application-metrics/example-web-python-monitor/0 (It may take up to a minute for Prometheus to load the new configuration and scrape the metrics).

## Task 8.1.2: Deploy a database and use a sidecar container to expose metrics

Task description:

As we’ve learned in Lab 4 - Prometheus exporters when applications do not expose metrics in the Prometheus format, there are a lot of exporters available to convert metrics into the correct format. In Kubernetes this is often done by deploying so called sidecar containers along with the actual application.

Use the following command to deploy a MariaDB database in the application-metrics namespace.

`curl -o ~/work/mariadb.yaml \`

`https://raw.githubusercontent.com/puzzle/prometheus-training/main/content/en/docs/08/labs/mariadb.yaml`

`kubectl -n application-metrics apply -f ~/work/mariadb.yaml`


This will create a Secret (username password to access the database), a Service and the Deployment .

> - Deploy the mariadb exporter from https://registry.hub.docker.com/r/prom/mysqld-exporter/ as a sidecar container
>> - Alter the existing MariaDB deployment definition (~/work/mariadb.yaml) to contain the side car
>> - Apply your changes to the cluster with kubectl -n application-metrics apply -f ~/work/mariadb.yaml
> - Create a ServiceMonitor to instruct Prometheus to scrape the sidecar container

First we need to alter the deployment of the MariaDB with adding the MariaDB exporter as a second container. Then extend the service by adding a second port for the MariaDB exporter.

>
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: mariadb
    labels:
        app: mariadb
        prometheus-monitoring: 'true'
    spec:
    ports:
        - name: mariadb
        port: 3306
        protocol: TCP
        targetPort: 3306
        - name: mariadb-exp
        port: 9104
        protocol: TCP
        targetPort: 9104
    selector:
        app: mariadb
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: mariadb
    labels:
        app: mariadb
    spec:
    selector:
        matchLabels:
        app: mariadb
    strategy:
        type: Recreate
    template:
        metadata:
        labels:
            app: mariadb
        spec:
        containers:
            - image: quay.io/prometheus/mysqld-exporter:latest
            name: mariadb-exporter
            env:
                - name: MYSQL_USER
                valueFrom:
                    secretKeyRef:
                    key: database-user
                    name: mariadb
                - name: MYSQL_PASSWORD
                valueFrom:
                    secretKeyRef:
                    key: database-password
                    name: mariadb
                - name: MYSQL_ROOT_PASSWORD
                valueFrom:
                    secretKeyRef:
                    key: database-root-password
                    name: mariadb
                - name: MYSQL_DATABASE
                valueFrom:
                    secretKeyRef:
                    key: database-name
                    name: mariadb
                - name: DATA_SOURCE_NAME
                value: $(MYSQL_USER):$(MYSQL_PASSWORD)@(127.0.0.1:3306)/$(MYSQL_DATABASE)
            ports:
                - containerPort: 9104
                name: mariadb-exp
            - image: mariadb:10.5
            name: mariadb
            args:
                - --ignore-db-dir=lost+found
            env:
                - name: MYSQL_USER
                valueFrom:
                    secretKeyRef:
                    key: database-user
                    name: mariadb
                - name: MYSQL_PASSWORD
                valueFrom:
                    secretKeyRef:
                    key: database-password
                    name: mariadb
                - name: MYSQL_ROOT_PASSWORD
                valueFrom:
                    secretKeyRef:
                    key: database-root-password
                    name: mariadb
                - name: MYSQL_DATABASE
                valueFrom:
                    secretKeyRef:
                    key: database-name
                    name: mariadb
            livenessProbe:
                tcpSocket:
                port: 3306
            ports:
                - containerPort: 3306
                name: mariadb


We can apply the file above using:

`kubectl -n application-metrics apply -f ~/work/mariadb.yaml`


Then we also need to create a new ServiceMonitor ~/work/servicemonitor-sidecar.yaml.

>
    ---
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
    labels:
        app.kubernetes.io/name: mariadb
    name: mariadb
    spec:
    endpoints:
        - interval: 30s
        port: mariadb-exp
        scheme: http
        path: /metrics
    selector:
        matchLabels:
        prometheus-monitoring: 'true'


`kubectl -n application-metrics apply -f ~/work/servicemonitor-sidecar.yaml`


Verify that the target gets scraped in the Prometheus user interface . Target name: application-metrics/mariadb/0 (It may take up to a minute for Prometheus to load the new configuration and scrape the metrics).

## Task 8.1.3: Troubleshooting Kubernetes Service Discovery

We will now deploy an application with an error in the monitoring configration.

> - Deploy Loki in the loki namespace

`kubectl create ns loki`

`kubectl -n loki create deployment loki \`

`--image=mirror.gcr.io/grafana/loki:latest`


> - Create a Service for Loki

`kubectl -n loki create -f \`

`https://raw.githubusercontent.com/puzzle/prometheus-training/main/content/en/docs/08/labs/service-loki.yaml`


> - Create the Loki ServiceMonitor

`kubectl -n loki create -f \`

`https://raw.githubusercontent.com/puzzle/prometheus-training/main/content/en/docs/08/labs/servicemonitor-loki.yaml`


> - When you visit the Prometheus user interface you will notice, that the Prometheus Server does not scrape metrics from Loki. Try to find out why.

## Troubleshooting: Prometheus is not scrapping metrics

The cause that Prometheus is not able to scrape metrics is usually one of the following.

The configuration defined in the ServiceMonitor does not appear in the Prometheus scrape configuration
> - Check if the label of your ServiceMonitor matches the label defined in the serviceMonitorSelector field of the Prometheus custom resource
> - Check the Prometheus operator logs for errors (Permission issues or invalid ServiceMonitors)

The Endpoint appears in the Prometheus scrape config but not under targets.
> - The namespaceSelector in the ServiceMonitor does not match the namespace of your app
> - The label selector does not match the Service of your app
> - The port name does not match the Service of your app

The Endpoint appears as a Prometheus target, but no data gets scraped
> - The application does not provide metrics under the correct path and port
> - Networking issues
> - Authentication required, but not configured

The quickest way to do this is to follow the instructions in the info box above. So let’s first find out which of the following statements apply to us

> - The configuration defined in the ServiceMonitor does not appear in the Prometheus scrape configuration
>> - Let’s check if Prometheus reads the configuration defined in the ServiceMonitor resource. To do so navigate to Prometheus configuration and search if loki appears in the scrape_configuration. You should find a job with the name serviceMonitor/loki/loki/0, therefore this should not be the issue in this case.

The Endpoint appears in the Prometheus configuration but not under targets.
Lets check if the application is running

`kubectl -n loki get pod`


The output should be similar to the following:
>
    NAME                    READY   STATUS    RESTARTS   AGE
    loki-5846d87f4c-tthsr   1/1     Running   0          34m


Lets check if the application is exposing metrics

`PODNAME=$(kubectl get pod -n loki -l app=loki -o name)`

`kubectl -n loki exec $PODNAME -it -- wget -O - localhost:3100/metrics`

`...`


The application exposes metrics and Prometheus generated the configuration according to the defined servicemonitor. Let’s verify, if the ServiceMonitor matches the Service.

`kubectl -n loki get svc loki -o yaml`

>
    apiVersion: v1
    kind: Service
    metadata:
    ...
    labels:
        app: loki
    name: loki
    namespace: loki
    spec:
    ...
    ports:
    - name: http
        ...


We see that the Service has the port named http and the label app: loki set. Let’s check the ServiceMonitor

`kubectl -n loki get servicemonitor loki -o yaml`

>
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    ...
    spec:
    endpoints:
    - interval: 30s
        ...
        port: http
        ...
    selector:
        matchLabels:
        prometheus-monitoring: "true"


We see that the ServiceMonitor expect the port named http and a label prometheus-monitoring: "true" set. So the culprit is the missing label. Let’s set the label on the Service.

`kubectl -n loki label svc loki prometheus-monitoring=true`


Verify that the target now gets scraped in the Prometheus user interface .

## Task 8.1.4: Blackbox monitoring in Kubernetes (optional)

In Lab 4 - Prometheus exporters we came across the blackbox exporter and learned how configuring a multi-target exporter through relabel_configs can be a bit tricky to understand. The Prometheus operator brings us a so-called Probe custom resource, which allows us to define the targets for a black box exporter in a much simplified way.

Task description:

> - Create a Probe custom resource in the application-metrics namespace for the example-web-python application
> - Use the Prometheus expression browser to check if the new metric is being scraped

Note: Use kubectl describe crd probe | less to describe the crd and get the available options.

Create the following probe custom resource (~/work/probe.yaml) in the application-metrics namespace

>
    ---
    apiVersion: monitoring.coreos.com/v1
    kind: Probe
    metadata:
    name: example-web-python-probe
    spec:
    module: http_2xx
    prober:
        url: blackbox-exporter.monitoring.svc:19115
    targets:
        staticConfig:
        static:
            - example-web-python.application-metrics.svc:5000/health


Apply it using the following command:

`kubectl -n application-metrics apply -f ~/work/probe.yaml`


Verify that the target gets scraped in the Prometheus user interface . Target name: application-metrics/example-web-python-probe (It may take up to a minute for Prometheus to load the new configuration and scrape the metrics).

Check for the following metric in Prometheus:

`{instance="example-web-python.application-metrics.svc:5000/health"}`
***