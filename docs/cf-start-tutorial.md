# Introduction

As CF operators you will be called on to dive into the CF platform. For example to track down issues pertaining to the operation of Application Instances deployed on your CF foundation. In this short tutorial we aim to give you some tools with which you can begin your forensic analysis, and also highlight what to look for in the logs. We will be using a newly deployed application to make this easier to reason about. 

The goal of this tutorial is to show what to look for when determining that an app is properly up and running on CF.

# But first, let's define a few things

### Auctioneer
This is the component responsible for holding placement auctions for any type of workload CF accepts onto a Diego Cell and placing the workload onto the winning Cell.

### BBS:
Bulletin Board System is the central data store and orchestrator of a Diego cluster. It communicates via protocol-buffer-encoded RPC-style calls over HTTP.

### Cell Rep: 
Application that runs on the Diego Cells and that serves as Representative of the Cell in auctions. It accepts work assigned to it by the Auctioneer and runs and schedules it to run. 

### Droplet

An archive within Cloud Foundry that contains the application ready to run on Diego. A droplet is the result of the application staging process.

### Garden: 
Application that exposes an API to manage and create containers with plug-able backends for Linux and Windows.

### Workload
Workloads are type of processes that are able to run on the CF, currently there exists two types of workloads: 
- **L**ong **R**unning **P**rocesses (**LRP**) 
- Tasks

### LRP (**L**ong **R**unning **P**rocess): 
Long Running Process is user issued process scheduled to run for an infinite amount of time. For example a long running server application.

### Task: 
Running process that will run in a finite amount of time. For example cron jobs

# Let's begin: A room with a view (starting with the CF CLI)

For the purpose of this tutorial we recommend you use the our [sample http app](https://github.com/cloudfoundry/sample-http-app) but it is possible for you to use any application that you know to work with CF. 

Make sure your [CF is up, running and operational](https://docs.cloudfoundry.org/deploying/index.html), and use the [CF CLI](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html) to push your application onto your desired ORG and SPACE.

To begin our analysis we'll need to gather a few unique identifiers:
- **AppGUID** ( the application's unique identifier on the CF platform)
- **Cell ID** ( the unique identifier of the cell on which our app is depoyed )
- **Instance GUID** ( unique identifier of the version of our application )

For a freshly pushed and started app the following command should provide us will all that we need to capture the **AppGUID**, **Cell ID** and the **Instance GUID**:

`cf logs <app name> --recent`

the output should look like:
```{.line-numbers}
1.  |   2018-05-29T10:49:32.26-0700 [API/0] OUT Created app with guid <AppGUID>
2.  |   2018-05-29T10:49:33.10-0700 [API/0] OUT Uploading bits for app with guid <AppGUID>
3.  |   2018-05-29T10:49:39.40-0700 [API/0] OUT Creating build for app with guid <AppGUID>
4.  |   2018-05-29T10:49:39.77-0700 [API/0] OUT Updated app with guid <AppGUID> ({"state"=>"STARTED"})
5.  |   2018-05-29T10:49:39.88-0700 [STG/0] OUT Downloading binary_buildpack...
6.  |   [... information pertaining to staging ommited ...]
7.  |   2018-05-29T10:50:13.60-0700 [CELL/0] OUT Cell <Cell ID> creating container for instance <Instance GUID>
8.  |   2018-05-29T10:50:14.29-0700 [CELL/0] OUT Cell <Cell ID> successfully created container for instance <Instance GUID>
9.  |   2018-05-29T10:50:16.58-0700 [CELL/0] OUT Starting health monitoring of container
10. |   2018-05-29T10:50:18.12-0700 [APP/PROC/WEB/0] [... omitted ...]
11. |   2018-05-29T10:50:19.15-0700 [CELL/0] OUT Container became healthy
```
> You can also find the app guid by issuing `cf app <app name> --guid`

In our example output, we've omitted most of the staging output, demarked by `[STG/*]` and the application's output demarked by `[APP/PROC/*/*]`

- At line __1__ we can find the **AppGUID**
- At line __7__ we can find the **Cell ID** and the **Instance GUID**
- Line __11__ shows us that our application is up and running

# Into the breach: Perspective from the BBS 

Now that we've determined our app is running, from the CF CLI's perspective, let's examine what went on from the BBS's perspective. We'll need the appguid in order to filter out the logs that are important to us. And because we do not know where the logs are located, rather we cannot be sure that the BBS that is currently the master is the node that has the logs for our app. We will make use of bosh's ability to dispatch a single command to multiple nodes to gather the info we require. 

With the previously obtained appguid issue the following:
`bosh -d cf ssh diego-api -c "zgrep \"<AppGuid>\" /var/vcap/sys/log/bbs/*"`

> we make use of `zgrep` in order to be able to search into zipped files incase the logs were turned over.

You should have obtained a series of log lines, let's take a look at a cleaned up version of these log lines. ( we used `jq .message` to obtain the lines below.)

> Not all lines have been included only the most significant will appear

#### The Diego API is called to request a Desired LRP

```
"bbs.request.desire-lrp.starting"
"bbs.request.desire-lrp.complete"
```

#### BBS auctions off work for the new Desired LRP sending the request to the Auctioneer

```
"bbs.request.desire-lrp.start-instance-range.create-unclaimed-actual-lrp.starting"
"bbs.request.desire-lrp.start-instance-range.create-unclaimed-actual-lrp.starting"
"bbs.request.desire-lrp.start-instance-range.create-unclaimed-actual-lrp.complete"
"bbs.request.desire-lrp.start-instance-range.start-lrp-auction-request"
"bbs.request.desire-lrp.start-instance-range.finished-lrp-auction-request"
```

#### Rep claims Actual LRP

```
"bbs.request.claim-actual-lrp.starting"
"bbs.request.claim-actual-lrp.complete"
```

#### The Cell Rep tells the BBS that the LRP has started

```
"bbs.request.start-actual-lrp.starting"
"bbs.request.start-actual-lrp.completed"
```

# Journey at the center of the REP (Understanding what goes on on the  REP)

Finaly let's get the Cell Rep's perspective on what went on. For that we should `SSH` into the Diego Cell with the Cell ID by issuing: `bosh -d cf ssh diego-cell/<Cell ID>` Then navigate to `/var/vcap/sys/log/rep` 

Look for the instance id to see what is going on 

 `grep -h "<Instance ID>" rep.std* | grep -v bulker | jq .message`
 
 The output should look like the lines below:
 > In the output we've filtered out the `bulker` logs as they are periodic and recurring and also only show the `message` portion of the logs
 
#### Work is placed on the rep:
> For clarity we've omitted `rep.request.auction-perform-work.auction-work` at the start of each line and replaced it with `*`
```
"*.lrp-allocate-instances.requesting-container-allocation"
"*.lrp-allocate-instances.succeeded-requesting-container-allocation"
```

#### Rep requests container from executor:

```
"rep.executing-container-operation.starting"
"rep.executing-container-operation.ordinary-lrp-processor.process-reserved-container.running-container"
"rep.executing-container-operation.ordinary-lrp-processor.process-reserved-container.succeeded-running-container"
"rep.executing-container-operation.finished"
```

#### Executor creates the container and downloads lifecycle binaries if needed
> For clarity we've omitted `rep.executing-container-operation.ordinary-lrp-processor.process-reserved-container` at the start of each line and replaced it with `*`
```
"*.run-container.creating-container"
"*.run-container.containerstore-create.starting"
"*.run-container.containerstore-create.node-create.cached-dependency-rate-limiter"
[...]
"*.run-container.containerstore-create.node-create.downloader.file-cache.get-directory.finished"
"*.run-container.containerstore-create.node-create.downloader.download.starting"
[...]
"*.run-container.containerstore-create.node-create.downloader.download.completed"
[...]
"*.run-container.containerstore-create.node-create.downloader.file-cache.add-directory.finished"
[...]
"*.run-container.containerstore-create.node-create.creating-container-in-garden"
"*.run-container.containerstore-create.node-create.created-container-in-garden"
"*.run-container.containerstore-create.complete"
"*.run-container.succeeded-creating-container-in-garden"
```

#### Executor downloads the instance dependencies (e.g. app droplet) and runs setup work in container
> For clarity we've omitted `rep.executing-container-operation.ordinary-lrp-processor.process-reserved-container.run-container` at the start of each line and replaced it with `*`
```
"*.running-container-in-garden"
"*.containerstore-run.starting"
[...]
"*.containerstore-run.complete"
"*.succeeded-running-container-in-garden"
[...]
"*.containerstore-run.node-run.cred-manager-runner.generating-credentials.complete"
"*.containerstore-run.node-run.cred-manager-runner.started"
[...]
"*.containerstore-run.node-run.setup.download-step.downloader.download.copy-to-destination-file.copy-finished"
[...]
"*.containerstore-run.node-run.setup.download-step.fetch-complete"
[...]
"*.containerstore-run.node-run.setup.download-step.stream-in-complete"
```

#### Executor starts applications process
> For clarity we've omitted `rep.executing-container-operation.ordinary-lrp-processor.process-reserved-container` at the start of each line and replaced it with `*`
```
"*.run-container.containerstore-run.node-run.action.run-step.running"
"*.containerstore-run.node-run.readiness-check.run-step.running"
"*.containerstore-run.node-run.action.run-step.running"
"*.containerstore-run.node-run.readiness-check.run-step.process-exit"
"*.containerstore-run.node-run.health-check-step.transitioned-to-healthy"
"*.containerstore-run.node-run.liveness-check.run-step.running"
"rep.executing-container-operation.starting"
"rep.executing-container-operation.ordinary-lrp-processor.process-running-container.bbs-start-actual-lrp"
"rep.executing-container-operation.finished"
```

# Conclusion (It's only the beginning...)

In conclusion, we've seen how a starting app looks like from the logging perspective of 3 different parts of a CF install:
- CF CLI
- Diego BBS
- Cell REP

this should give you a good idea of what to look for when a healthy application has started and is considered healthy. You should also now have a better idea on how to leverage the bosh cli  to get at that information. In a next installment we will look at how to get a view of the system using the `cfdot`.
