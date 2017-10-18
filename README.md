UKWA Documentation
==================

Technical documentation for the UK Web Archive.

<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [Introduction](#introduction)
- [System Overview](#system-overview)
- [Services & Repositories](#services--repositories)
	- [Services](#services)
	- [Management](#management)
	- [Monitoring](#monitoring)
	- [Major Components & Interfaces](#major-components--interfaces)
- [Further Information](#further-information)

<!-- /MarkdownTOC -->

<!--
See the [ukwa-documentation](https://github.com/ukwa/ukwa-documentation#ukwa-documentation) for an overview.
-->


Introduction
------------

This document outlines the current and near-future web archiving system. It is intended to give an introduction to the major components and tie together the different source code repositories that are involved.


System Overview
---------------

High-level workflows are described here - more detailed documentation on individual tasks will be available [here](http://ukwa-shepherd.readthedocs.io/en/latest/).

![High-level Architectural overview](./drawings/ng-was.png)

The goal is a stable and robust crawl lifecycle, automated end-to-end. The frequent crawl stream should run with no manual intervention, and while the domain crawl may be initiated manually, the ingest and processing of content collected by the crawl should use the same framework.

During long-running processes, failure is expected. Therefore, everything is built to be idempotent, as it may be retried many times. Transient failure are reported but not critical unless persistent.

Overall, we use a small number of larger, modular components (mostly written in Java), and then ‘glue’ them together using Python scripts. Where appropriate, these scripts use the [*Luigi*](https://github.com/spotify/luigi) task framework to help ensure the processes are robust and recoverable. All services are wrapped as Docker containers for deployment.

On the *ingest* side, we can make use of any crawl engine that:

* Can consume a list of seeds and honor an appropriate scope:
    * Either the 'crawl feed' from W3ACT that defines our frequent crawls (seeds, caps, crawl launch dates and frequencies, etc.)
    * Or the domain crawl seed list, with a scope that permits GeoIP lookup as well as U.K. domain scoping.
* Emits the results in the form we require:
    * A set of WARC files, and associated logs.
    * In a standard folder structure reflecting the crawl stream and the crawl launch date.
    * Along with any necessary metadata e.g. crawl configuration files, usually packaged as a ZIP.

This material is cached on HDFS, from where it can be indexed for access and also wrapped for submission to the DLS. All critical data is stored here as simple files in the standard folder structure, and all irreplaceable data should be store in the same way. All downstream processes depend only upon the state of the HDFS system (either directly or in faster, cached forms). i.e. there should be no coupling between the *ingest* and *access* sides.

Hadoop jobs are used to index the WARC files for access and search, and WARC records can be streamed from the HDFS storage backend as required. The indexes and HDFS provide the primary APIs that power the front-end services.

All *ingest* and *access* tasks are launched via cron from a central service (known as `sh`), but each is scheduled by a dedicated instance of the Python Luigi Daemon. For example, a cron job on `sh` may initiate a Luigi task on the the *ingest* server that scans crawl engines for new content to upload to HDFS.

An entirely independent *monitor* layer runs it's own tasks to inspect the status of the services and the various data stores, and stores the results in ElasticSearch for inspection in Grafana and/or Kibana. We recognise the [monitoring and QA are the same thing](https://plus.google.com/+RipRowan/posts/eVeouesvaVX) and aim to raise the bar for automated monitoring an QA of web archiving processes.


Services & Repositories
-----------------------

<table width="100%" style="text-align: center; display: table; border-collapse: collapse;">
	<thead width="100%">
	    <tr><th></th><th>Ingest</th><th>Storage</th><th>Access</th></tr>
    </thead>
    <tbody width="100%">
		<tr>
			<th rowspan="2">Services</th>
			<td><a href="https://github.com/ukwa/ukwa-ingest-services"><i>ukwa-ingest-services</i></a></td>
			<td>HDFS, HBase</td>
			<td><a href="https://github.com/ukwa/ukwa-access-services"><i>ukwa-access-services</i></a></td>
		</tr>
		<tr>
			<td>On Docker</td>
			<td>Native</td>
			<td>On Docker</td>
		</tr>
		<tr>
			<th rowspan="2">Management</th>
			<td colspan="3" align="center"><a href="https://github.com/spotify/luigi"><i>Python Luigi</i></a> tasks defined in <a href="https://github.com/ukwa/ukwa-manage"><i>ukwa-manage</i></a></td>
		</tr>
		<tr>
			<td><i>Ingest Task Scheduler</i></td>
			<td></td>
			<td><i>Access Task Scheduler</i></td>
		</tr>
		<tr>
			<th rowspan="2">Monitoring</th>
			<td colspan="3" align="center">Tasks in <a href="https://github.com/ukwa/ukwa-monitor"><i>ukwa-monitor</i></a></td>
		</tr>
		<tr>
			<td colspan="3" align="center">Monitoring Task Scheduler</td>
		</tr>
	</tbody>
</table>


There are three major areas to cover: Services, Management & Monitoring. The Services are the main components of the crawl system, usually deployed as Docker containers. The Management system coordinates the tasks that act upon the services, in order to automate whole content management life-cycle. The Monitoring system runs independently of the Management system, but automates checks that help ensure the whole system is running correctly.

### Services ###

There are two main sets of services, each of which has one or more [*Docker Compose*](https://docs.docker.com/compose/) files to define different deployments, from local development to full production.

- [*ukwa-ingest-services*](https://github.com/ukwa/ukwa-ingest-services) with separate Docker Compose definitions for the crawl engine (Heritrix3 etc.) and for the frontend services (e.g. W3ACT etc.) 
- [*ukwa-access-services*](https://github.com/ukwa/ukwa-access-services) which covers end-user services like our website, APIs etc.

### Management ###

The [*ukwa-manage*](https://github.com/ukwa/ukwa-manage) code base contains all the ‘glue’ code that orchestrates our content life-cycle management, implemented as [*Python Luigi*](https://github.com/spotify/luigi) tasks.

All events, across all production systems, are initiated by cron jobs on `wash`. For example, a cron job on `wash` may initiate a Luigi task on the the `ingest` server that scans crawl engines for new content to upload to HDFS. Both the `ingest` and `access` servers have a copy of `python-shepherd` installed, and each runs their own LuigiD task scheduler (which provides a UI and manages task execution).

n.b. we may split the management codebase into a library and three separate task sets (for ingest, storage and access) in the future.

### Monitoring ###

The [*ukwa-monitor*](https://github.com/ukwa/ukwa-monitor) codebase monitors and reports on all UKWA processes, including ingest. This provides HTML reports, a dashboard, and raises alerts by probing the production system. It is also based on Luigi tasks, but no part of any production system depends on it, and it runs it’s own cron jobs (rather than being initiated by `wash`) and it's own LuigiD scheduler. If it's down, nothing else should be affected. 

The monitoring tasks check that the necessary services are running, and that critical management tasks have been executed successfully. If services are down, or transient failures become persistant, it is the role of the monitor engine to raise the alert.

### Major Components & Interfaces ###

The curation front end, the frequent crawl engine, the shepherd, the domain crawler, the storage, the indexes, the access services.


- Developed by us and/or IIPC members:
    - [*W3ACT*](https://github.com/ukwa/w3act) (UKWA-only)
    - [*Heritrix3*](https://github.com/internetarchive/heritrix3)
    - [*warcprox*](https://github.com/internetarchive/warcprox) (Python)
    - [*webarchive-discovery*](https://github.com/ukwa/webarchive-discovery)
    - [*OutbackCDX*](https://github.com/nla/outbackcdx) (a.k.a. tinycdxserver)
    - [*OpenWayback*](https://github.com/iipc/openwayback)
    - The UKWA Website engine [*ukwa/ukwa-ui*](https://github.com/ukwa/ukwa-ui)


Further Information
-------------------

* [Design Principles](Design-Principles.md)
* [System State Management](System-State-Management.md)
* [Deployment](Deployment.md)
* [Development](Development.md)
    * [Working With Luigi](Working-With-Luigi.md)
* [Future Development](Future-Development.md)
