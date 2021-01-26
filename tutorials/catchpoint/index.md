# Google Cloud Platform Network Performance Monitoring using Catchpoint

## Overview

Catchpoint digital experience monitoring tools provide instant insights into the performance of networks, apps, and digital services. Customers gain visibility with network telemetry metrics for ISP to Cloud Endpoints or Virtual Machine SaaS Endpoints. The data Catchpoint collects aids diagnosing issues, faster times to recover, defining accurate SLAs, and general performance troubleshooting.

This guide walks through multiple approaches for ingesting Catchpoint data into Google Cloud Platform.

# Rework
- how to enable enhanced capabilities to leverage BigQuery for querying the large data set metrics
- how to realize the vision of a single pane for all network performance monitoring and analysis

## Catchpoint Supported Test Types

- Synthetic Monitoring from any location
  - Create transactional tests to replicate [Real User Monitoring](https://en.wikipedia.org/wiki/Real_user_monitoring#:~:text=Real%20user%20monitoring%20(RUM)%20is,server%20or%20cloud%2Dbased%20application.&text=The%20data%20may%20also%20be,intended%20effect%20or%20cause%20errors).
- Network Layer Tests from any location
  - Layer 3 (traceroute, ping), Layer 4 (TCP, UDP), DNS and BGP.

For more information, please refer to the [Catchpoint Platform](https://www.catchpoint.com/platform) documentation.

This guide highlights two approaches to consume the Catchpoint test data within GCP

# Need to fix these links, share the files

1. [Creating a GCP Data Ingestion Pipeline for Catchpoint Test data](https://docs.google.com/document/d/1KuIc-A45aFJ3eK-Rm46nyf_bsR_TvPD2MhXU9Qd_Z_U/edit?ts=600b384e#heading=h.96l8hfgcpy6x)
2. [Catchpoint and Cloud Monitoring Integration](https://docs.google.com/document/d/1KuIc-A45aFJ3eK-Rm46nyf_bsR_TvPD2MhXU9Qd_Z_U/edit?ts=600b384e#heading=h.p33702x5820f)

## Creating a GCP Data Ingestion Pipeline for Catchpoint Test Data

### Pipeline Architecture

![The end-to-end data ingestion pipeline within GCP. Catchpoint to App Engine to Cloud Pub/Sub to Cloud Dataflow to BigQuery to Grafana](integration-pipeline.png)

The steps from Catchpoint to Grafana

1. Catchpoint sends data via a predefined webhook. The template is configured within the Catchpoint portal (see next section for more details)
2. Webhook runs in GCP using App Engine. **THIS IS WEIRD and i don't know what it means**
3. App Engine uses [Pub-Sub](https://cloud.google.com/pubsub) to propagate data to configured channels.
4. A [Dataflow](https://cloud.google.com/dataflow/) job listens to the Pub/Sub channel and inserts the a BigQuery accessible dataset.
5. Data is processed to match the Catchpoint Schema and sent to [BigQuery](https://cloud.google.com/bigquery).
6. Grafana uses a BigQuery data source to visualize the data.

### Configuration Details

This section covers each configuration step with further explanations to set up each component and example scripts. The configuration details describes two main tasks

# Need to fix these links, share the files
1. [GCP Pipeline Setup](https://docs.google.com/document/d/1KuIc-A45aFJ3eK-Rm46nyf_bsR_TvPD2MhXU9Qd_Z_U/edit?ts=600b384e#heading=h.ytup525q5dqj)
1. [Creating the Catchpoint Setup](https://docs.google.com/document/d/1KuIc-A45aFJ3eK-Rm46nyf_bsR_TvPD2MhXU9Qd_Z_U/edit?ts=600b384e#heading=h.xpxkt04y4bh4)

#### GCP Pipeline Setup

##### Create a Pub/Sub topic for streaming data into the ingestion pipeline

Before App Engine deployment, a Pub/Sub topic is needed to receive and distribute data. Pub/Sub enables data to be ingested from one data source and streamed to another data source. For this tutorial, the Publisher topic hosts the data subject (push data) and the subscriber to the topic will forward data to a streaming service.

[How to create a Topic](https://cloud.google.com/pubsub/docs/quickstart-console#create_a_topic)

1. Create a Publisher topic
![create-pub-topic](create-pub-topic.png)

1. Create a Subscription to receive messages
![create-subscription](create-subscription.png)

##### Build a Webhook in GCP

A webhook (web-application) is needed to post data from vendors. The app will listen on the defined URL and is used to push the data out to a Pub/Sub created in the above step.

To create the `/cppush` testing webhook used in this example, use the **go** script in the GCS bucket [here](https://storage.googleapis.com/webhook-catchpoint/main.go).

Once downloaded, change the following configuration values:

```config
DefaultCloudProjectName: <your-project-id>
CatchpointTopicProd: <pub-sub-topic-name>
CatchpointPushURL: <URL> # in this example it is /cppush
```

[App Engine Deployment reference](https://cloud.google.com/appengine/docs/standard/go/building-app#deploying_your_web_service_on)

#### BigQuery dataset and data tables

Once the Pub/Sub topics are setup and data is flowing, BigQuery needs the data to run analytics. Before the pipeline can be created, the BigQuery table needs to be setup.

[How to create a BigQuery table](https://cloud.google.com/bigquery/docs/tables)

##### Create a BigQuery dataset

![create-big-query](create-big-query.png)

To create the main and dead_letter tables, refer to the documentation for the [main table](https://storage.cloud.google.com/netperf-bucket/CatchPoint%20-%20main%20table?cloudshell=true) and [dead_letter table](https://storage.cloud.google.com/netperf-bucket/Dead_Letter%20table?cloudshell=true).

##### Build an Ingestion Pipeline

In this example, Cloud DataFlow is used to input data. The DataFlow code can be found [here](https://github.com/pupamanyu/beam-pipelines/tree/master/perf-data-loader).

To match the Catchpoint test data schema, `metric.java` needs to be changed. A sample Catchpoint `metric.java` file can be found [here](https://storage.cloud.google.com/netperf-bucket/CatchPoint%20-%20metric.java).

Once `metric.java` is changed, the job is ready to be deployed  
_Note: This java build requires Java8. To switch to Java8 on cloud shell run this command_

```sh
sudo update-java-alternatives \
  -s java-1.8.0-openjdk-amd64 && \
  export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
```

To build the Fat Jar, execute the below command from within the project root directory

```sh
./gradlew clean && ./gradlew shadowJar
```

To run the pipeline, replace the values then execute the below command from within the project root directory.

```sh
cd build/libs && java -jar perf-data-loader-1.0.jar \
  --dataSet=<target-dataset> \
  --table=<target-table> \
  --deadLetterDataSet=<dead-letter-dataset> \
  --deadLetterTable=<dead-letter-table> \
  --runner=DataflowRunner \
  --project=<gcp-project-name> \
  --subscription=projects/<gcp-project-name>/subscriptions/<pub-sub-subscription> \
  --jobName=<pipeline-job-name>
```

To update the pipeline, execute the below command from within the project root directory.

```sh
$ cd build/libs && java -jar perf-data-loader-1.0.jar \
  --dataSet=<target-dataset> \
  --table=<target-table> \
  --deadLetterDataSet=<dead-letter-dataset> \
  --deadLetterTable=<dead-letter-table> \
  --runner=DataflowRunner \
  --project=<gcp-project-name> \
  --subscription=projects/<gcp-project-name>/subscriptions/<pub-sub-subscription> \
  --jobName=<existing-pipeline-job-name>
  --update
```

###### A sample deployed job

![sample-deployed-job](sample-deployed-job.png)

##### Configuring Grafana

Grafana is available in the GCP market place. A guide to deploy Grafana in your GCP project is available [here](https://console.cloud.google.com/marketplace/details/click-to-deploy-images/grafana).

Grafana offers a plugin for BigQuery as a data source. Step by step instructions are available [here](https://grafana.com/grafana/plugins/doitintl-bigquery-datasource).

###### Sample Grafana dashboards

![grafana-dashboard-1](grafana-dashboard-1.png)

![grafana-dashboard-2](grafana-dashboard-2.png)

![grafana-dashboard-3](grafana-dashboard-3.png)

#### Catchpoint setup

After the GCP ingestion pipeline is built, CatchPoint tests must be configured to post data to the GCP webhook.

##### Create a Catchpoint WebHook

1. Navigate to [Catchpoint API Detail](https://portal.catchpoint.com/ui/Content/Administration/ApiDetail.aspx). See [Catchpoint Webhook document](https://support.catchpoint.com/hc/en-us/articles/115005282906) for additional information.  
![portal-api-menu](portal-api-menu.png)

1. Select **Add Url** under **Test Data Webhook**  
![webhook-new](webhook-new.png)

    i. Under "URL" add the url for the webhook application created in the previous section
    ii. Under "Format" select "Template Radio Button"
    iii. Click "Select Template"
    iv. Click "Add New"  
    ![webhook-configure-template](webhook-configure-template.png)

1. Provide a name and select "JSON" as the format.
    - See the [Test Data Webhook Macros](https://support.catchpoint.com/hc/en-us/articles/360008476571) for options
    - Sample JSON template containing recommended macros

    ```json
    {
        "TestName": "${TestName}",
        "TestURL": "${testurl}",
        "TimeStamp": "${timestamp}",
        "NodeName": "${nodeName}",
        "PacketLoss": "${pingpacketlosspct}",
        "RTTAvg": "${pingroundtriptimeavg}",
        "DNSTime": "${timingdns}", 
        "Connect": "${timingconnect}", 
        "SSL": "${timingssl}", 
        "SendTime": "${timingsend}",
        "WaitTime": "${timingwait}", 
        "Total": "${timingtotal}"
    }
    ```

1. Click "Save" at the bottom of the page.

### Catchpoint and Cloud Monitoring Integration

Cloud Monitoring provides visibility into the performance, uptime, and overall health of applications. It collects metrics, events, and metadata from Google Cloud, Amazon Web Services, hosted uptime probes, application instrumentation, and a variety of common application components including Cassandra, Nginx, Apache Web Server, Elasticsearch, and many others. Operations ingests that data and generates insights via dashboards, charts, and alerts. Cloud Monitoring alerting helps you collaborate by integrating with Slack, PagerDuty, and more.

This guide provides steps to configure ingesting data from Catchpoint into [Google Cloud Monitoring](https://cloud.google.com/monitoring)

#### Simplified Data Ingestion Pipeline

![data-ingestion-pipeline](data-ingestion-pipeline.png)

##### Configuration Steps

1. Create a new project in Google Console or reuse an existing project

1. Enable Monitoring API

1. Enable Cloud Functions

1. GCP data pipeline setup

1. Cloud Monitoring set up

1. Catchpoint set up

1. Create Dashboards and Metric Explorer in Cloud Monitoring

##### Integration Configuration Details

1. Create a new project in Google Console or reuse an existing project  
Google Cloud projects form the basis for creating, enabling, and using all Google Cloud services including managing APIs, enabling billing, adding and removing collaborators, and managing permissions for Google Cloud resources. Refer to [Creating and managing projects | Resource Manager Documentation](https://cloud.google.com/resource-manager/docs/creating-managing-projects) for steps to create a new project.  
_The Project id is used to configure the Catchpoint Integration script._

1. Enable Monitoring API  
The Monitoring API must be enabled and have authorized users. Follow the steps in [Enabling the Monitoring API | Cloud Monitoring](https://cloud.google.com/monitoring/api/enable-api) to enable and authorize use of the Monitoring API v3. The Monitoring API can be enabled using either the Cloud SDK or the Cloud console. Both these approaches have been explained in the referenced guide.

1. Enable Cloud Functions  
Follow the steps in [Cloud Pub/Sub Tutorial | Cloud Functions Documentation](https://cloud.google.com/functions/docs/tutorials/pubsub) to enable the use of Cloud Functions and Cloud Pub/Sub APIs. This tutorial leverages Node.js. The referenced guide contains information to setup your development environment.

##### GCP data pipeline setup

1. Clone the Catchpoint Stackdriver integration repository to your local machine from [here](https://github.com/catchpoint/Integrations.GoogleCloudMonitoring). In this example, we will be using the push model to ingest data from Catchpoint to Cloud Monitoring. The Stackdriver-Webhook folder has the required Node.js script to set up the ingestion of data and writing of data to Cloud monitoring.  
Set the GoogleProjectId environment variable by using the ID from step1. Update the GoogleProjectId variable in the .env file at [https://github.com/catchpoint/Integrations.GoogleCloudMonitoring/blob/master/Stackdriver-Webhook/.env](https://github.com/catchpoint/Integrations.GoogleCloudMonitoring/blob/master/Stackdriver-Webhook/.env). Refer to [Using Environment Variables | Cloud Functions Documentation](https://cloud.google.com/functions/docs/env-var) for information on how to set up environment variables.

`index.js` snippet to be used below.

```javascript
'use strict';
const monitoring = require('@google-cloud/monitoring');
const { PubSub } = require('@google-cloud/pubsub');
const path = require('path')
const dotenv = require('dotenv')
dotenv.config({ path: path.join(\_\_dirname, '.env') });
const pubsub = new PubSub();
const googleProjectId = process.env.GoogleProjectId;
```

1. Creating cloud functions to stream data to an ingestion pipeline
1. Open Google Cloud SDK Shell and navigate to the directory where the Node.js scripts were cloned  
    `cd <path to cloned directory>`
1. To set/change the project property in the core section, run:  
   `gcloud config set project my-project`

1. Next, we will deploy two cloud functions to handle data injection from Catchpoint. We will be leveraging Pub/Sub, an asynchronous messaging service that decouples services that produce events from services that process events. Refer to [https://cloud.google.com/pubsub/docs/overview](https://cloud.google.com/pubsub/docs/overview) for understanding more about how these functions are useful.

# REWORK THIS SECTION
In this integration, we are using Catchpoint's ability to push data to a webhook.  This is the recommended approach for ingesting Catchpoint data into Cloud monitoring as it gives you the flexibility of adapting to new metrics being added on Catchpoint side and also ensure you get data in real time from the Catchpoint platform.

###### TIP: Using Catchpoint REST APIs

_In case you need to leverage the PULL model to integrate with Catchpoint, you can look at the example that leverages Catchpoint REST APIs in the Github repository_ [_here_](https://github.com/catchpoint/Integrations.GoogleCloudMonitoring)_. The catchpoint-rest-api.js has the code to leverage the Catchpoint performance API to pull data from Catchpoint. If you use this approach, you can skip the cloud function step up and go over to the Cloud Monitoring Set Up._

1. HTTP triggers are used to invoke cloud functions with an HTTP request. Additional information is available [here](https://cloud.google.com/functions/docs/calling/http).  
The publish function gets triggered by a HTTP POST from Catchpoint. This function then publishes to the topic specified while deploying the subscriber.

###### The Publish function

```javascript
/**
Publishes a message to a Google Cloud Pub/Sub Topic.
*/

exports.catchpointPublish = async (req, res) => {
    console.log(`Publishing message to topic ${topicName}.`);
    const pubsub = new PubSub();
    const topic = pubsub.topic(topicName);
    const data = JSON.stringify(req.body);
    const message = Buffer.from(data, 'utf8');

    try {
        await topic.publish(message);
        res.status(200).send(`Message published to topic ${topicName}.`);
    } catch (err) {
        console.error(err);
        res.status(500).send(err);
        return Promise.reject(err);
    }
};
```

##### Deploy publish function

1. Run the below command  

    ```sh
    gcloud functions deploy catchpointPublish \
        --trigger-http \
        --runtime nodejs10 \
        --trigger-http \
        --allow-unauthenticated
    ```

2. Copy the URL after the deploy is successful. This is needed to set up the Catchpoint Webhook in Step 6.1.

![sample-publish-function](sample-publish-function.png)

1. Pub/Sub Trigger  
Cloud Functions can be triggered by messages published to [Pub/Sub topics](https://cloud.google.com/pubsub/docs) in the same Cloud project as the function. More information on calling Pub/Sub Topics is available [here](https://cloud.google.com/functions/docs/calling/pubsub).  

###### The Subscribe function

The subscriber function subscribers to the topic to which we publish the Catchpoint data. It then feeds it to Stackdriver.

```javascript
/*
Triggered from a message on Google Cloud Pub/Sub topic.
*/

exports.catchpointSubscribe = (message) => {
    const data = Buffer.from(message.data, 'base64').toString();
    const catchpointData = JSON.parse(data);
    postToGoogleMonitoring(catchpointData);
};
```

##### Deploy Subscribe function

1. Run the below command  
_Note: Either create the topic separately before deploying or alternatively specify it directly while deploying. If the topic is not created, this command automatically creates a new topic._  

    ```sh
    gcloud functions deploy catchpointSubscribe
        --trigger-topic catchpoint-webhook
        --runtime nodejs10
        --allow-unauthenticated
    ```

![sample-subscribe-function](sample-subscribe-function.png)

1. Set the TopicName environment variable using the same topic name you used to deploy the Subscribe function. Update the TopicName variable in the .env file at [https://github.com/catchpoint/Integrations.GoogleCloudMonitoring/blob/master/Stackdriver-Webhook/.env](https://github.com/catchpoint/Integrations.GoogleCloudMonitoring/blob/master/Stackdriver-Webhook/.env). Refer to [https://cloud.google.com/functions/docs/env-var](https://cloud.google.com/functions/docs/env-var) for information on how to set up environment variables.  

    ```sh
    gcloud pubsub topics create <topic-name>
    ```

    The topic will be referenced in `index.js` as below

    ```javascript
    const topicName = process.env.TopicName;
    ```

##### Cloud Monitoring set up

To use Cloud Monitoring, you must have a Google Cloud project with billing enabled. The project must also be associated with a Workspace. Cloud Monitoring uses Workspaces to organize monitored Google Cloud projects.

In the Google Cloud Console, go to Monitoring -> Overview. This will create a workspace for you automatically for the first time.

![gcm-menu](gcm-menu.png)

We will be using the Monitoring Client Libraries to write data to Cloud Monitoring in index.js. Please refer to [Monitoring Client Libraries | Cloud Monitoring](https://cloud.google.com/monitoring/docs/reference/libraries) for instructions on installing the client library and setting up authentication. You must complete this before leveraging them in the Node.js script.

The postToGoogleMonitoring function in index.js handles parsing of the data posted from Catchpoint, constructs a timeseries object and writes this data to Cloud Monitoring.

Catchpoint gives you the ability to collect end user performance data from their probes deployed in multiple Countries, Cities and ISPs across the globe. Consider a Catchpoint monitor that monitors a webpage. Catchpoint collects metrics like DNS, Connect, SSL, Wait(TTFB), Load , Time To Interactive(TTI), First Paint and many more. These metrics help you triangulate issues. For example, let's say the Time to Interactive (TTI) for a page increased, the additional metrics help you understand whether it is the DNS or the network causing the issue.

We wanted to have the benefit of correlation and posted all the key metrics from Catchpoint to Cloud Monitoring. Since Catchpoint metrics are not predefined in GCP, we are using the concept of custom metrics. Refer to [Creating custom metrics | Cloud Monitoring](https://cloud.google.com/monitoring/custom-metrics/creating-metrics)for more information on this.

If the metric name is DNS in Catchpoint, we are creating a custom metric called catchpoint_DNS in Cloud Monitoring.

The below snippet of code parses Catchpoint data and constructs a timeseries object

```javascript
for (var i = 0; i < metrics.length; i++) {
    let metricValue = response.Summary.Timing[metrics[i]];
    let dataPoint = parseDataPoint(metricValue);
    let metric = 'catchpoint\_' + metrics[i];
    
    timeSeriesData[i] = parseTimeSeriesData(metric, dataPoint, testId, nodeName);
}
```

We wanted to ensure that each data point is associated with the prober location and monitor ID in Catchpoint. These are specified as labels.

Below is the snippet of code that shows the construction of the time series data –

```javascript
function parseTimeSeriesData(metric, dataPoint, testId, nodeName) {
    const timeSeriesData = {
        metric: {
            type: 'custom.googleapis.com/global/' + metric,
            labels: {
                Test_id: testId,
                Node: nodeName
            },
        },
        resource: {
            type: 'global',
            labels: {
                project_id: googleProjectId,
            },
        },
        points: [dataPoint]
    };

    return timeSeriesData;
}
```

Finally, we use the `writeRequest` function to write the time series data into Cloud Monitoring. Below is the snippet of code that writes the data -

```javascript
const client = new monitoring.MetricServiceClient();
const writeRequest = {
    name: client.projectPath(googleProjectId),
    timeSeries: timeSeriesData
};
```

1. Catchpoint setup

1. Webhook set up:

Go to Settings -> API in Catchpoint

![portal-api-menu](portal-api-menu.png)

Under Test Data Webhook, click on Add Url. Enter the Webhook Url you got after deploying the publish function from Step 4.3.a in the URL section.

![test-data-webhook-new](test-data-webhook-new.png)

You can leverage the default JSON payload posted by Catchpoint or create a template with specific metrics similar to the BigQuery integration example.

1. Dashboards and Metric Explorer in Cloud Monitoring

Once all the above pieces are set up and the data starts flowing into Cloud Monitoring, you have the ability to use Metric Explorer to perform analysis and also create Dashboards.

To view the metrics for a monitored resource using Metrics Explorer, do the following:

1. In the Google Cloud Console, go to Monitoring  
   [Google Cloud Platform](https://console.cloud.google.com/monitoring)

1. In the Monitoring navigation pane, click on Metrics Explorer.

1. Enter the monitored resource name in the Find resource type and metric text box.

![gcm-metrics](gcm-metrics.png)  ![gcm-build-query-metrics](gcm-build-query-metrics.png)

1. Metrics explorer also allows to filter data points using the labels node name or test id.

![gcm-build-query-filters](gcm-build-query-filters.png)

1. Add all the required metrics and optionally save the chart to a dashboard. 
   [Metrics Explorer | Cloud Monitoring](https://cloud.google.com/monitoring/charts/metrics-explorer)

1. Navigate to Monitoring->Dashboards to check out the metrics. Here is a sample Dashboard that shows Catchpoint data in Cloud Monitoring –

![gcm-dashboard](gcm-dashboard.png)

While creating the charts, you have the ability to choose different visualizations and also select the required statistical aggregation. In the Dashboard above, we are looking at 95p.

Refer to [Creating charts | Cloud Monitoring](https://cloud.google.com/monitoring/charts) for more information on creating charts and Dashboards.