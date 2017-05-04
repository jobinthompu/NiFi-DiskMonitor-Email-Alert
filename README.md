# NiFi-DiskMonitor-Email-Alert

## Short Description

Sample NiFi Flow for Monitoring and alerting while MonitorDiskUsage or MonitorMemory Reporting Task WARNINGS are generated when the defined threshold exceeds. 

## Introduction

Recently I was asked how to Monitor and alert while MonitorDiskUsage or MonitorMemory Reporting Task WARNINGS are generated when it exceeds a predefined threshold, I am trying to capture the steps to implement the same.

## Prerequisites

1) To test this, Make sure HDF-2.x version of NiFi is up an running.

2) You Already have a MonitorDiskUsage/MonitorMemory Reporting task created and configured, set a lower threshold to force generate the alert while testing.

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/10.png)

3) Make a note of the Reporting Task uuid to be monitored:

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/0.png)

## Creating a Flow to Monitor Reporting Task.

1) Drop a GenerateFlowFile to execute every 5mins, with 1 byte file to trigger the flow.

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/1.png)

2) Drop an UpdateAttribute processor to the canvas with below configuration:

```
NIFI_HOST 			: <Your NiFi FQDN>
NIFI_PORT 			: <Your NiFi http port>
REPORTING-TASK-UUID : <Your MonitorDiskUsage or MonitorMemory controller service uuid>
```
![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/2.png)

3) Connect Success relationship of GenerateFlowFile to UpdateAttribute.

4) Drop a InvokeHTTP processor to the canvas, and configure it as below:
```
HTTP Method : GET
Remote URL :  http://${NIFI_HOST}:${NIFI_PORT}/nifi-api/reporting-tasks/${REPORTING-TASK-UUID}

```
![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/3.png)

5) Connect Success relation of UpdateAttribute  to InvokeHTTP and auto terminate all relationships of InvokeHTTP processor but Response relationship.

6) Drop a EvaluateJsonPath processor to the canvas with below configuration:

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/4.png)

7) Auto terminate All relationship of EvaluateJsonPath processor except Matched relationship and connect InvokeHTTP processor’s Response relation to it.

8) Drop a SplitJson processor to canvas with below Configurtaion. [The reason for splitting json is because the REST call to reporting task gives duplicate json array in the result, which contains details we need.]

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/5.png)

9) Connect Matched relationship of EvaluateJsonPath to SplitJson processor.

10) Drop another EvaluateJsonPath processor to the canvas with below configuration:

```
LEVEL 		:	$.bulletin.level
MESSAGE 	:	$.bulletin.message
SOURCE-NAME	:	$.bulletin.sourceName
TIMESTAMP 	:	$.bulletin.timestamp
```
![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/6.png)

11) Add a connection with split relationship from SplitJson to second EvaluateJsonPath processor. Auto-terminate other relationships.

12) Add a ControlRate processor so that only one alert is sent for multiple json arrays with below configuration which passes only 1 flowfile per minute.

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/7.png)

13) Add a connection from EvaluateJsonPath processor to ControlRate processor with FlowFile Expiration as 30 sec. Auto-terminate other relations.

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/8.png)

14) Finally Drop an PutEmail processor to canvas with below configuration to sent your alerts, update with your SMTP details

```
SMTP Hostname		:	west.xxxx.yourServer.net
SMTP Port			:	587
SMTP Username		:	jgeorge@hortonworks.com
SMTP Password		: 	Its_myPassw0rd_updateY0urs
SMTP TLS			:	true
From				:	jgeorge@hortonworks.com
To					:	jgeorge@hortonworks.com
Subject				:	${sourceName} ALERT
```

and message content should look something like below to grab all the values:

```
Message :				${MESSAGE}
						LEVEL		:	${LEVEL}
						TIMESTAMP		:	${TIMESTAMP}
						SOURCE-NAME	:	${SOURCE-NAME}
```
![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/9.png)

15) Auto terminate all relationships of PutEmail processor and connect Success relationship of ControlRate processor to it.

16) Now you have your flow ready and can start it to monitor and sent Email Alert for UI Notification thrown by NiFi when Disk or Memory utilization exceeds given threshold.


## Staring the flow, Reporting task and testing it

1) Now lets start the MonitorDiskUsage Reporting task to generate an Alert to test it with below configuration with a lower threshold to force generate the alert. I am monitoring disk on my mac and it’s disk is more than 50% utilized, so I will get this WARNING in the NiFi UI. 

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/11.png)

2) As soon as this Warning show up and the flow created is running you will get alert in your inbox stating the same, similar as below:

![alt tag](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/images/12.png)

3) This concludes the tutorial for Monitoring your Reporting Tasks with NiFi itself.

4) Too lazy to create the flow???.. Download my template [here](https://github.com/jobinthompu/NiFi-DiskMonitor-Email-Alert/blob/master/resources/REPORTING_TASK_ALERT.xml)

## References
[NiFi REST API](https://nifi.apache.org/docs/nifi-docs/rest-api/index.html)

[NiFI Expression Language](https://nifi.apache.org/docs/nifi-docs/html/expression-language-guide.html)

Thanks,

Jobin George


