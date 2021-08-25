:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

Introduction
============

The Rubin Observatory Data Facilities have three major roles in the operation of the Data Management System: they store raw data, they perform data processing (in particular, the Data Release Production), and they serve as Data Access Centers.
The US Data Facility (USDF) is constructed and operated by the project, while the French Data Facility (FrDF) and prospective UK Data Facility (UKDF) are constructed and operated by organizations in those countries.
Note that there will be other Data Access Centers that do not participate in data processing, such as the Chilean Data Access Center and other international Data Access Centers.

This document specifies the interfaces that the US Data Facility must provide to the applications that run there.
Other Data Facilities will require subsets of these interfaces, as there are some functions that are performed only by the USDF or by the USDF and FrDF, as enumerated in the subsections below.
Requirements are stated for the USDF and are adapted for other Data Facilities based on the subsets they implement.
The document expands upon the overall DM System design in LDM-148 by describing implementation choices that have been made.
It gives more specific requirements than those in LSE-61.
Sizing for computational infrastructure is given in DMTN-135.
Other Data Facilities must provide the same interfaces for data processing, but they may choose to implement their Data Access Center in a different way.


Common Functions
----------------

All Data Facilities store permanently some fraction of raw LSSTCam science data, currently 100% for the USDF and FrDF, 25% for the UKDF.
This fraction is determined by sky region, as almost all processing can be spatially localized.
They also store copies of all master calibration and global calibration data products.
All of this data, as well as intermediate scratch data products, will be ingested into a Data Butler at each site.

The Facilities provide some fraction of the compute power to execute the Data Release Production (DRP), currently 50% for the FrDF, 25% for the USDF and UKDF, as a batch service.
This also includes executing automated quality control processing alongside the DRP.

To enable transfer of raw data and data products between Data Facilities, they each host a Data Backbone (DBB) endpoint.

Each Facility will host a Data Access Center, running the Rubin Science Platform, but its contents may vary, with differing numbers of releases and differing selections of data products potentially targeted for a different user community.
The Qserv distributed database is expected to be used to enable query access to large catalogs, but it is not a mandatory component of a Facility.

USDF and FrDF
-------------

The USDF and FrDF will store permanently all of the raw science data from the LSST Atmospheric Transmission Imager and Slitless Spectrograph (LATISS), raw calibration images, the Engineering and Facility Database (EFD), and the Large File Annex (LFA).
They will also both store Commissioning data that is part of the permanent survey record, including data from the Commissioning Camera (ComCam).
THey will also store and archive all released data products, both images and catalogs, including the Data Releases themselves as well as the Prompt Products Database and the Alert Database.

These two Facilities serve as disaster recovery sites for each other.

USDF only
---------

The USDF maintains the Data Backbone central database, tracking all raw data and data product files.
It coordinates all Data Release Production processing, hosting the Campaign Management system.
It is currently anticipated that it will also host the Workflow Management system, with the other Facilities providing simple batch services.

It is exclusively responsible for Prompt Processing, including execution of the Alert Production and daily calibration processing, Solar System processing, the Alert Production Database (APDB), and Alert Distribution.
As a result, it is the endpoint for the dedicated international long-haul network from the Observatory.
It performs periodic calibration processing to generate master calibration data products that are then transferred to the other Facilities.
It also performs periodic template generation.

Commissioning, ComCam, and LATISS offline processing beyond the scope of the Chilean Commissioning Cluster will be executed at the USDF.

The USDF will host the display systems for quality control as well as the analysis and display systems for human-driven quality assurance, verification and validation, and system performance tracking and measurement.

Development clusters for maintenance and integration of new versions of Data Facility software and systems will be part of the USDF.

The USDF authentication systems will host the master list of data rights holders and their group memberships.

Users
-----

These Data Facilities support two main classes of users: "science users" and "staff users".
"Science users" have data rights as described in RDO-13.
They query, retrieve, and analyze data products using the Data Access Center facilities.
They (or intermediary brokers) may also subscribe to the stream of alerts from Alert Production.
"Staff users" are employees of Rubin Observatory or a Data Facility, or they are non-employee adjunct staff assisting with the Commissioning process, with verification and validation, or with quality assurance.
They build, maintain, debug, and repair the Data Facility systems and the scientific software that runs on them.
They have rights to look at partial and intermediate data products before they are released, using internal Data Facility systems that are not part of the Data Access Center.

Enclaves
--------

As described in LDM-148, the USDF can be divided into five enclaves:
* Prompt US Enclave (not present in other Data Facilities)
* Offline Production Enclave
* Archive US Enclave
* US Data Access Center
* Development and Integration Enclave (not present in other Data Facilities)

Each contains computational infrastructure that allow application services to be deployed and operated in the enclave.

Timelines
---------

The Data Facilities need to be operational in FY24, with preceding overlap with Construction operations to ensure a smooth transition to the start of Operations.

Requirements
^^^^^^^^^^^^

The USDF will

* provide the full-sized Development and Integration Enclave at least one year prior to the commencement of Operations
* provide Prompt, Archive, Offline Production, and Data Access Center Enclaves sized similarly to the Interim Data Facility at least one year prior to the commencement of Operations
* provide 100 Gbit/sec path-redundant network capacity between some point on the existing Rubin Long-Haul Network and the USDF at least 6 months prior to the commencement of Operations

The FrDF and UKDF will

* provide Archive and Offline Production Enclaves sized proportionally at least 6 months prior to the commencement of Operations
* provide sufficient network capacity between the USDF and FrDF and between the UKDF and FrDF at least 6 months prior to the commencement of Operations

Computational Infrastructure
============================

As described in LDM-148 §12, the infrastructure underlying all the enclaves includes database services, batch computing, and containerized application management.

The major components of the computational infrastructure are:

- Object storage and POSIX filesystems

- Databases (PostgresQL, Oracle at USDF only if PanDA is selected for workflow, Cassandra cluster at USDF only, InfluxDB at USDF and FrDF)

- HTCondor, Slurm, or PanDA batch service and underlying compute machines

- Kubernetes clusters

- Puppet/Terraform system configuration control

- Identity management

- IT security

- Service management and monitoring including log collection and analysis

Storage Services
----------------

Most raw data and data products are expected to be stored in object storage, as this is thought to be the most cost-effective way to provide scalable-bandwidth access to the very large amounts of write-once, read-many data required, but POSIX filesystems with suitable bandwidth are acceptable as an alternative, if available at lower cost, and some POSIX storage is expected to be desirable for datasets that are modified in place in the course of the DRP.
Science and staff users will also require POSIX filesystems for read/write data in user and shared group space.
Older datasets that are part of the permanent record of the survey will be preserved on archival storage.

Datasets in object storage will have three characteristic access patterns:

Data Release datasets
^^^^^^^^^^^^^^^^^^^^^

Datasets that are part of Data Releases are expected to be accessed frequently by science users.
Low latency is required in order to serve these rapidly to a large user base, although there are no quantitative retrieval performance requirements.
The most stringent use case is likely to be providing "movie-like" access to cutouts of processed visit images (PVIs).
If we assume lossy-compressed PVIs are about 32 MB per CCD image and that the movie is displayed at 10 frames per second, the bandwidth required is at least 320 MB/sec per retrieval and up to 1280 MB/sec if the cutout falls on an image corner.

Catalog datasets in Parquet form may also be frequently accessed for complex analyses that are not amenable to ADQL.
Since high-performance queries are intended to be satisfied by Qserv, the bandwidth requirements for catalog access should be modest, but it should be possible to scan all of the catalog files in a reasonable amount of time.
Since these files grow from 8 to 80 PB, if we assume a full scan can take up to 30 days, 3 to 30 GB/sec of bandwidth will be needed.

There may be "hot spots" for access to these datasets as multiple science users retrieve the same data (e.g. for a class, or because something interesting has happened in a particular location), but otherwise access is expected to be fairly uniformly spread across the available data, including some users who may attempt to scan across all of the data (at whatever speed their resource allocations allow).

DRP intermediates
^^^^^^^^^^^^^^^^^

Datasets that are intermediates in Data Release Production may be read multiple times during the course of the annual processing.
There might be 5 steps that access the intermediate and 10 spatial regions overlapping it in each step, for 50 accesses over its lifetime.
Intermediates are expected to be kept for most of the duration of the DRP in order to enable forensic investigation of any execution or data quality problems.
They will then be replaced by the next year's DRP intermediates.
Access patterns are fairly regular and can be precomputed based on the processing dependency graph.

Raw data
^^^^^^^^

Raw science and calibration images are expected to be read very infrequently, on the order of once or twice per year, as most science users will start with coadded images or PVIs.
These accesses are also regular and can be precomputed.

Durability
^^^^^^^^^^

Data Release datasets and raw images can be recovered from archival storage or another Data Facility if lost or corrupted, so the durability of these datasets does not need to be very high.
Intermediates can usually be regenerated, but doing so may require complex and time-consuming backtracking in the workflow, so their storage should be more reliable.

Requirements
^^^^^^^^^^^^

All DFs will

* provide scalable-size and scalable-bandwidth object storage, preferably emphasizing low cost over low latency
* provide scalable-size and scalable-bandwidth POSIX filesystems with locking capabilities (e.g. NFS v4), preferably emphasizing user-level performance at reasonable cost, including SSD where required

The USDF and FrDF will

* provide high-latency backup of permanent record datasets (e.g. on tape)
* develop, practice, and use (when necessary) disaster recovery processes

Network Services
----------------

The design for the networks linking the observatory and the Data Facilities is described in LSE-78.
The long-haul network must operate with not only high bandwidth but also low total latency, within a 3 sec budget for transfers of up to 3 GB from the Summit.
External bandwidth to community broker partners must be at least 10 Gbit/sec as specified in LDM-612.
The USDF-FrDF link is provided by ESnet with contributions from GEANT and RENATER, but the USDF must not impede its full capacity.
The internal network within the Data Facility is not specified, but capacities of 100 Gbit/sec to each processing machine and large aggregate bandwidths to storage are expected.
Kubernetes clusters may require specific network overlays to support features required by the services deployed on them.

Most staff-accessible machines should be behind a bastion/jump host or VPN.
Public Internet ingress, defined by the relevant deployment, will be required to the DAC and selected other services.

Requirements
^^^^^^^^^^^^

The USDF will

* manage and maintain the endpoint for the international long-haul network and interface with the network team to ensure uptime and performance

All DFs will

* manage and maintain the inbound and outbound networks to partners
* manage and maintain the internal networks within the Facility, including Kubernetes overlays

Database Services
-----------------

Database services used by each application in an enclave are currently expected to be deployed independently as part of the application.
This provides maximum resilience and independence.
However, for efficiency of management and performance, it may make sense to consolidate several databases of similar technology into a single database management system.
In most cases, PostgreSQL is the preferred database technology, although Oracle, Cassandra, InfluxDB, and other NoSQL technology (not yet selected) may be required.

One major user of database services is the Data Butler Registry.
This database tracks all data products used by and produced by Science Pipelines processing.
It is currently anticipated that each Data Facility participating in Data Release Production will have its own local subset of the Butler Registry.
This subset will be updated by pipeline jobs that execute at that Facility.
The Facilities will then exchange Registry information as they exchange data products.
The USDF and FrDF copies of the Registry will eventually contain metadata and provenance for all data products.

Requirements
^^^^^^^^^^^^

All DFs will

* deploy, manage, and administer the central database systems
* perform periodic backups and restores as needed
* provide and maintain the underlying computer systems and storage
* assist with the use of database-level mechanisms for authentication, authorization, and resource quota enforcement where required

The USDF will

* advise and consult on schema design, schema migration, and performance tuning
* serve as an example for configuring databases at the other Data Facilities

Batch Computing
---------------

Batch computing is currently specified to use HTCondor.
Plugins to the batch computing services may allow other back-end batch systems with equivalent functionality (such as Pegasus, CWL, or PanDA) to be used, but those are still under development.
Batch computing may be implemented as a traditional submission/script-driven system, but it would be preferable to be able to execute containers.
Security concerns would have to be taken into account, as batch jobs are expected to execute under science user accounts in addition to production service accounts.
This may necessitate the use of restricted containers such as Singularity, at least for non-production queues.
Batch computing for science users will require access to their per-user and shared project POSIX filesystems, but it may also require access to data in object store via VO services or via the Butler client.
Separate deployments could be used in each enclave, or each enclave could be served with dedicated queues in an overall shared batch cluster, with priorities as given in LDM-148 §12.1.2.

Almost all batch computing may be run on shared or preemptible machines, as this is intended for high-throughput computing.
(It may be desirable to have a small queue for non-preemptible critical jobs.)
Batch jobs are expected to execute in less than 24 hours, with most completing in a few hours, even after grouping of jobs to minimize overhead.
At present, GPUs are not used by any of the LSST processing pipelines.
If GPUs are shown to provide sufficient speedup to significantly impact the overall DRP execution cost, it is possible that they will be incorporated in the future, at least for some jobs.
Memory usage varies, with most jobs (but not necessarily compute time) executing in 5-6 GB per core but some jobs currently requiring 20-30 GB per core.


The amount of batch processing needed will be elastic; DRP needs will vary during the course of the annual processing, and science users will have bursts of usage, particularly around the time of the Data Release or key conferences.
Accordingly, being able to automatically grow and shrink the processing clusters would be desirable.

Requirements
^^^^^^^^^^^^

All DFs will

* deploy, manage, and administer the HTCondor system(s)
* provide and maintain the underlying computer systems and storage
* assist with the use of batch-system-level mechanisms to enforce resource quotas where required

Containerized Application Management
------------------------------------

Containerized application management will use Kubernetes to deploy application containers in pods.
The prime application is the Rubin Science Platform within the Data Access Center, but there are many other applications in this and other enclaves that will be deployed on Kubernetes.
Examples range from QA plots with drill-down analysis to Qserv.
Again, separate deployments per enclave can be used or a single shared deployment with appropriate namespacing and resource management could be used.
Most applications are expected to be deployed using ArgoCD.
The batch computing system above need not be deployed this way, although PanDA, for example, can be deployed on top of Kubernetes.
Containers will be published to public registries, but it will likely be useful to have a local cache within each DF to improve performance and avoid rate limits.

Requirements
^^^^^^^^^^^^

All DFs will

* deploy, manage, and administer the Kubernetes cluster(s)
* provide and maintain the underlying computer systems and storage
* assist with the use of Kubernetes mechanisms to enforce resource quotas where required
* provide a local container registry capable of being operated in "cache" replication mode

ITC Provisioning and Management
-------------------------------

The USDF needs to procure, install, provision, maintain, and retire computational infrastructure in a timely fashion.
Appropriate, documented, well-practiced processes for these steps must be in place.

The use of infrastructure-as-code to configure the complex compute, storage, and network components in each enclave is highly desirable.
Tools such as Puppet and Terraform are expected to be used to accomplish this.
It is recommended that the USDF embrace an automated Git-Ops deployment model for infrastructure.

Requirements
^^^^^^^^^^^^

All DFs will

* develop, document, and practice procurement, provisioning, maintenance, and retirement processes for computational infrastructure
* ensure that all system configurations are recorded and that deployments are reproducible

Identity Management
-------------------

SQR-044 documents the requirements for identity management for the Rubin Science Platform within the US Data Access Center.
Of note is that RSP accounts are expected to be managed directly by RSP systems and administrators, using RSP-specific policies.
Integration with host system (SLAC or cloud provider) accounts is seen as being a loose association, if present at all.
We would prefer that no such association is required.
If host system accounts are required, the process for obtaining them must be highly automated, at least for US scientists.

Requirements
^^^^^^^^^^^^

All DFs will

* ensure that the requirements of SQR-044 are met

IT Security
-----------

The USDF is expected to provide information security in accordance with LPM-121 and LPM-122.
Note that the raw images are classified as Internal, with a more stringent level of protection, until they are published as Prompt Data Products, at which point they become classified as Protected User.
It is anticipated that the raw images would not be transferred to partner Data Facilities until that publication point.
Prior to that transfer, the redundant image replicas are located in the Camera DAQ system at the Summit, the Chilean Archive Enclave at the Base, and the USDF.

The Rubin Science Platform services (see below) currently expose an underlying POSIX filesystem to the science user for storage of User-Generated Data Products consisting of personal, group-shared, and public files.
These are distinct from the Project-generated Prompt and Data Release Data Products.
Authorizing access to files in that filesystem is handled using standard POSIX mechanisms (user ids, group ids, permissions, access control lists).
As a result, the RSP services that access the filesystem must impersonate the user, therefore requiring them to be started with privilege (which they can then give up).
It must be possible to create filesystem-visible accounts for users as they are added to the Science Platform.
At some point in the future, this storage may be intermediated by services such as VOSpace or WebDAV, which would remove the necessity for impersonation and filesystem-visible accounts, but this is not yet possible.

It must be possible for non-DF employees to be granted administrative roles to assist in the management of storage, Kubernetes, and the batch system, as well as to have access to the logging and monitoring services.
All administrative access by DF or Rubin staff requires two-factor authentication.

Requirements
^^^^^^^^^^^^

The USDF will

* secure its systems against appropriate threats without unduly interfering with RSP service needs
* ensure that RSP services that require privileges can be deployed and execute safely
* ensure that non-SLAC employees can act as administrators of appropriate systems
* ensure that Rubin can manage its own domains and associated SSL certificates, potentially using non-USDF services

Service Management/Monitoring
-----------------------------

Requirements
^^^^^^^^^^^^

All DFs will

* oversee service management processes such as change, release, and configuration management
* ensure that upgrades to hardware and service infrastructure can occur in a rolling fashion whenever feasible
* provide incident and request response processes for infrastructure systems
* provide monitoring of infrastructure and application services, including dashboard views of key system metrics and automated alerts to DF and Rubin engineers
* provide collection of system and application logs into a system that allows for monitoring and forensic analysis of incidents


Prompt US Enclave
=================

This enclave is responsible for hosting the Prompt Processing, Alert Distribution, and Prompt Quality Control services.

To support Prompt Processing, Alert Distribution and Prompt Quality Control, a cluster of compute nodes is required.
This cluster grows only as the complexity of the Alert Production software increases; it does not have increasing data loads, as the size of the camera and the area covered on the sky do not change.

Storage resources within the enclave primarily include local compute node filesystems and the internal Alert Production Database (APDB).
Other storage is accessed from the Archive Enclave, including its Butler repositories.
The APDB is expected to be implemented using a substantial Cassandra cluster with high-speed SSD storage; its size and configuration are being determined.
There is a ramp in the size of the APDB during the first year, but it then reaches steady state.

Parts of Prompt Processing may execute as batch jobs while other parts execute as long-running services on a Kubernetes cluster.
In both cases, the latency requirements of Prompt Processing make it desirable to maintain state local to the machine, such as master calibration images, across multiple jobs or multiple service requests.
Reloading that state for each image or visit may not be practical in the time available.
Prompt processing production will always execute under a service account, rather than as a particular science user.
The control system for Prompt Processing may require its own database to track its execution.

Parts of Prompt Processing execute during the daytime, handling Solar System processing and reprocessing or catch-up.
The Solar System processing (DMTN-087) involves integration with the Minor Planet Center (MPC), transmitting new candidate discoveries and receiving updated catalogs.

While the maximum compute requirements for Prompt Processing are generally fixed, there may be minor variations that require some elasticity in the cluster size.
For example, in areas of the sky with more objects, the Alert Production may need additional time and/or compute to perform its measurements.
This may cause a temporary need for additional compute nodes to handle the next (or next-plus-one) visit while the previous one completes.
In the other direction, daytime calibration processing may not be as resource-intensive as nightly Alert Production processing, allowing resources to be freed for other uses.

A Kafka deployment is required for Alert Distribution.

The long-haul international network connection from Chile is also part of this enclave.

Data flows for this enclave include raw data input (estimated at 3 GB every 15 seconds during both nighttime observation and daytime calibration, sourced from the long-haul network and temporarily stored on SSD before processing), retrieval of master calibration images and datasets (at most 19 GB every 30 seconds, but should be much less due to local caching, sourced from Archive Enclave object store), retrieval and storage of APDB data (at most 1 GB every 30 seconds but typically much less, sourced from the APDB), and generation of alerts (multiple streams of up to 1 GB every 30 seconds, sourced from Alert Generation).
The MPC interactions are relatively modest in comparison.

The Prompt Processing system has not been fully designed, but it should look something like this:

- The ``nextVisit`` event sent by the Script Queue on the Summit triggers the execution of pipelines at the USDF, one pipeline per CCD.

- These pipelines can preload data (including from the APDB) and precompute values based on the known boresight coordinates and predicted visit timespan.  They then block waiting for the images to arrive in the Data Butler.

- The CCS image writer system writes a FITS file including headers from the Header Service to its local storage in the Diagnostic Cluster.

- This file is passed to a script that uploads it directly from the Summit to object storage at the USDF.

- After the upload is complete, the script or an object-store notification triggers ingest of the image into the Data Butler registry at the USDF, allowing the pipelines to continue.

- The pipelines generate alerts and dispatch them to Kafka.

- Daytime Solar System processing is executed as a scheduled job.

- Daily calibration processing, if necessary, is executed manually or under (extended) OCPS control at the USDF.  Master calibration data products are certified by a human operator in the Data Butler registry at NCSA and in the Observatory Operations Data Service (OODS) at the Summit.  The Active Optics System and other observing systems can obtain them from there.


Offline Production Enclave
==========================

To support Batch Processing, which executes the Data Release Production, Calibration Products Production, Template Generation, offline Special Programs such as deep-drilling field processing documented in DMTN-065, and other offline processing needs, a large cluster of compute nodes is required.
This cluster grows with time as more processing power is needed to reprocess all the images taken to date for each Data Release.
Batch processing, as its name suggests, executes as batch jobs.
These production processes will always execute under a service account rather than as an individual user.

Storage interfaces within the enclave primarily include local compute node filesystems.
There may also be shared filesystems that contain intermediate data products that require in-place modification.
These node-local and shared scratch filesystems would be needed in each Data Facility.
We are expecting to store all read-only raw data and final data products on object store within Archive Enclave Butler repositories.

Campaign management and Workload/workflow management services run in this enclave.
Ideally both these levels will execute at the USDF with appropriate jobs being selected by the workflow system to be sent to other Data Facilities, but it may be necessary to have campaign management submit workflows to local services at each Data Facility.
The PanDA workflow service currently being evaluated to run under the Batch Production Service requires an Oracle database for high performance on large workflows, but there is the possibility of investing resources to improve its performance on PostgreSQL.
Other implementation options will likely use either Oracle or PostgreSQL.
These services will be deployed as containers in a Kubernetes cluster.

A DRP will begin with a specification of the software, pipeline and algorithm configurations, and inputs, including master calibration data products, that are to be used.
Software will be made available to each site via containers (or, if necessary, eups binary packages).
Raw images (for a fraction of the sky in the case of the UKDF) and master calibration data products will have already been ingested into the Data Butler at each site.
Each site will execute single-frame processing jobs in parallel for raw images in its assigned part of the sky, writing intermediate results back to local scratch and via the Data Backbone to the USDF where needed.
Automated quality control jobs will also execute along with the main processing, reporting metrics to a central display system at the USDF.
Global calibration steps will then execute at the USDF based on all of the collected intermediates, with result datasets sent back to the other Data Facilities via the Data Backbone.
The remainder of the processing, again localized by sky region, whether for coadd patches or further visit processing, will also be distributed across Data Facilities with results communicated to the USDF.
If any final global steps are required, they would again be performed at the USDF.
See DMTN-172 for more details on what the DRP pipelines are expected to look like.
Overall quality assurance is expected to be performed at the USDF, including characterization of the Data Release and feedback to System Performance, but there are opportunities for the European DFs to contribute to monitoring of the DRP execution by detecting problems and adjusting systems as necessary.
Final result data products will be collected at the USDF and shared with the FrDF as they are generated (for backup purposes).
(Of course, data products generated at the FrDF need not make a round-trip.)
All desired release data products will be shared from the USDF (or potentially the FrDF, if agreed) to other Data Access Centers, including the UKDF, prior to the Data Release so that all DACs can release the data at the same time.

Prior to each Data Release, at least one "mini-DRP" will be executed on a fraction of the sky as a "dress rehearsal" for the annual processing.
If successful, the mini-DRP outputs can be incorporated into the DRP without recalculation.

A Qserv distributed database instance will be hosted in this enclave.
Catalog data products will be exchanged between Data Facilities, with at least the USDF and French Data Facility receiving all catalog products.
This exchange is expected to occur in batches as workflows complete within the larger-scale Production, with the primary constraint being the ability to synchronize Butler Registry information which should be more performant for bulk updates rather than incremental ones.
As catalog datasets are generated or received, they will be incrementally loaded into the Qserv instance, and staff will have access to perform quality assurance.
(Qserv allows simultaneous loading and querying with consistent query results, except for a short pause while each ingest "super-transaction" is committed.)
Note that replication between sites is at the catalog dataset level, not a Qserv-to-Qserv replication.
At the time of Data Release, the contents of this database (or potentially the entire deployment itself) will be transferred to the US Data Access Center for use by science users.

Other quality control and quality assurance services (e.g. the SQuaSH metric timeline system and web servers hosting diagnostic plots as generated by the pipe_analysis package or its replacement) will also be located in this enclave.


Archive Enclave
===============

The Archive Enclave contains the permanent record of the Survey.
It primarily hosts storage in the form of object storage and databases.
All data products are backed up to archival storage (tape or other long-latency, low-cost storage).

The object storage holds raw data and data products.
It must provide an S3-like interface (or equivalent, with available, well-maintained, portable client libraries) as well as the ability to generate direct HTTP URLs for access to objects.
Those URLs should be able to be pre-signed to allow authorized access (read or write) to restricted objects.
At least parts of the Large File Annex (LFA), which stores files such as all-sky camera images or calibration spectrograph data emitted by Observatory systems, will also be hosted in this object storage; some (but not all) of its objects will be ingested into the Data Butler.
Objects that are not designated for the Data Butler may be left at the Summit only or may be incorporated into the Data Backbone only.

The Prompt Products Database (PPDB) (LDM-612, §2.3.3) is expected to be a relational (PostgreSQL) database.
It grows with time, as does the Alert Database (LDM-612, §2.3.5), which may be implemented simply as an object store or possibly as a NoSQL database.
The Engineering and Facilities Database is an InfluxDB instance, replicating one at the Summit via Kafka, along with an additional replica in relational (PostgreSQL) form suitable for joining with other relational tables.
Additional databases will be present, including the "Observatory Wide Logbook" (DMTN-173), a "live" exposure metadata database, and ancillary provenance information (e.g. from campaign management).

The Data Backbone maintains the files/objects in this enclave, managing the lifetime, replication, and backup to archival storage of all permanent survey datasets.
It includes Bulk Distribution/Download capabilities (DMTN-147) to replicate both raw and data product datasets to partner institutions, including other Data Facilities and Data Access Centers.

These bulk capabilities will include inbound (e.g. to retrieve data products or backups from other Data Facilities) and outbound (e.g. to provide data products to partners) network resources.

The Data Butler provides access to the enclave contents.
Its deployment includes a large shared relational database management system for its Registry and Datastore, anticipated to be implemented on PostgreSQL.
It will also include a Web service (DMTN-169) to provide authorized access to the Registry.
Primary provenance of datasets will be available from the Registry.
Production and user queries to this database will be minimized by the middleware, but the query load may still be high.

Certain data products are only available from archival storage: lossless-compressed processed images from all Data Releases (lossy-compressed for the most recent Data Releases is on disk) and Parquet catalogs from past Data Releases.
Retrieval of these is expected to be slow and infrequent, possibly even scheduled in advance via administrative request.


US Data Access Center
=====================

The US Data Access Center hosts a Rubin Science Platform (RSP) instance.
This includes the Notebook Aspect, a JupyterHub service providing notebooks and command-line-based interactive and batch compute access to science users; the Portal Aspect, currently an instance of the Firefly query and visualization tool; and the Web APIs Aspect, a set of VO-compliant Web services providing program-level access to data products.
Identity management and token-based authorization components are also part of this deployment.
Notebook, command-line, and batch computing for science users through the RSP is done as the user with appropriate UID and GIDs for each interactive or batch process.
The Web APIs are expected to execute under service accounts without user impersonation, implementing authorization internally.

A Qserv distributed database deployment provides the high-performance catalog query back-end required by the VO TAP service for large LSST datasets.
Note that this Qserv instance is considered part of the DAC and not the Archive Enclave; the file-based catalogs are the permanent record while Qserv is an application into which they are loaded.
Smaller datasets and metadata about all datasets such as TAP Schema information are stored in PostgreSQL.
Users will also be able to create their own personal User Generated tables in a relational database, including tables within Qserv to facilitate joining.
While these user tables are typically linked with a particular Data Release catalog, the existence of join tables that map between identifiers from different Data Releases means that they can continue to be useful with future Data Releases and so need to be preserved.
Other operational databases (e.g. the set of issued tokens in Redis) are deployed on a per-application basis.

As mentioned above, the RSP provides a "home directory" space in a POSIX filesystem to each user for file-oriented User Generated datasets as well as access to shared space for groups that the user is a member of.
The user and group filesystems require POSIX UID and GID authorization and enforceable quotas managed by UID or GID.
The RSP is designed to provision the per-user "home directory" dynamically when a new user is authorized to use the system (which requires privileged access), but it is possible to bypass this if provisioning is guaranteed through an external process.

The VO services and Data Butler Registry server in the US DAC will intermediate access by science users to all the contents of the Archive Enclave, including querying of the EFD, LFA, and other ancillary databases.


Development and Integration Enclave
===================================

The USDF provides resources for the Rubin Construction and Operations teams to develop and maintain software components of the Data Management System.
These include an interactive cluster with machines on which to execute the edit/build/test development cycle and a batch cluster for larger-scale testing.
Storage includes per-user home directories as well as shared, self-managed project and scratch spaces.
Storage also includes more-permanent storage for test datasets, including precursor data, simulated data, and selected subsets of real data.

Separate Kubernetes clusters enable integration and testing of containerized services.
These include one cluster for a "stable" deployment of the RSP as a Science Validation instance for staff to access the Development and Integration storage as well as the Offline Production and Archive Enclaves, another cluster for "integration" deployments of RSP components, and a final small "admin" cluster for testing new versions of Kubernetes itself.

The USDF will host a continuous integration build service (ci.lsst.codes) including both Linux and macOS build platforms.
A build artifact distribution server (eups.lsst.codes, object store with Web server front-end) with ingress from the Internet at large is also required.

Requirements
^^^^^^^^^^^^

The USDF will

* provide a team to build and maintain the Qserv distributed database
* contribute to middleware development during Construction and maintenance during Operations, including the Data Backbone and Batch Production Services
* enable a smooth transition from the NCSA LDF and Google Cloud IDF to the USDF for all transferred functions


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
