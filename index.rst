:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. _introduction:

Introduction
============

This tech note describes the uses of Data Management (DM) Science Pipelines code in observatory operations, including the computing and storage systems available to it as Commissioning proceeds.

.. _use-cases:

Use Cases
=========

DM Science Pipelines code has always been expected to produce Prompt data products and telemetry feedback to the Observatory Control System.
But it is now expected to be present in several other operational systems to perform:

* ISR of wavefront sensor images for the active optics system (AOS).
* Processing of full-focal-plane intra-focal and extra-focal images for the AOS.
* Centroiding of guider images.
* Initial spectrum processing to control the LSST Auxiliary Telescope Imager and Slitless Spectrograph (LATISS) positioning and focus.
* Automatic rapid analysis and visualization of images in the Camera Diagnostic Cluster.
* Ad hoc and scripted Commissioning-related analyses of images and telemetry.

In addition, the Science Pipelines environment is used as the basis for Telescope and Site Software (TSSW) Commandable SAL Component (CSC) development.

The `next section <satisfying-use-cases>`_ describes how these use cases are handled.
The Commissioning use case, because of its complexity and significance is described in its `own section <commissioning-analytics>`_.
The `following sections <summit-resources>` give more details about the services and resources available to execute the use cases.


.. _satisfying-use-cases:

Satisfying the Use Cases
========================

.. _satisfying-active-optics:

Active Optics
-------------

For more details, see the `Active Optics Data Flows <https://confluence.lsstcorp.org/x/SQfKBg>`_ Confluence page.

Wavefront sensor images from LSSTCam for the AOS, during normal science operations, are delivered by a copy of the `Camera Control System's (CCS) <ccs>`_ `Data Acquisition (DAQ) Image Driver <ccs-daq-driver>`_ resident on an `AOS machine <tssw-machines>`_ at the Summit.
The driver triggers ingestion into a `Butler repository <butler-repo>`_ local to that machine and also sends an "image available" event via `SAL <sal>`_ that includes the necessary information to form a Butler DataId.
The AOS processing code retrieves the image via the Data Butler.
Any master calibrations needed for image signature removal (ISR) are copied by a separate process from the `Data Backbone (DBB) <dbb>`_ and ingested into the Butler repository.

For full-frame processing with ComCam and LSSTCam, an AOS script is expected to control the entire system.
It sends "take images" commands via the `ScriptQueue CSC <script-queue>`_ to the CCS.
the science pixels travel via the `Archiver CSC <archiver>`_ to the `Observatory Operations Data Service (OODS) <oods>`_, both at the Base.
The Archiver sends an "image available" event via SAL that includes the necessary information to form a Butler DataId; this information is also available from the CCS via SAL messages.
The AOS then uses the `OCS-Controlled Batch <ocs-batch>`_ service to process the data.
The batch-executed pipelines use the Data Butler to retrieve images from the OODS.
Any master calibrations needed are also obtained from the OODS.


.. _satisfying-guider-centroiding:

Guider Centroiding
------------------

Guider region-of-interest (ROI) images are obtained via a custom Camera DAQ client using a special guider interface.
The image pixels in memory are converted to an `lsst.afw.Image` or `lsst.afw.Exposure` and passed to centroiding routines in the LSST Science Pipelines Libraries.
If ISR is required, the master calibrations are pre-loaded from a local Butler repository.

.. _satisfying-auxtel-control:

Auxiliary Telescope Control
---------------------------

There are a few alternatives here.

The baseline is to have images be captured by the CCS.
It triggers ingestion into a Butler repository local to that machine and also sends an "image available" event via SAL that includes the necessary information to form a Butler DataId.
The AuxTel control machine NFS mounts this repository and retrieves images from it via the Data Butler.
In addition, the `Summit Analysis machine <summit-analysis>`_, which provides rapid analysis in a notebook environment, will NFS mount the repository.
Any master calibrations needed for ISR are copied by a separate process from the DBB and ingested into the Butler repository.

One alternative would be to have this process be executed by the ATArchiver and an AuxTel OODS instance running on the same machine, both at the Summit.
The advantage of this would be that the images would get full Header Service headers and would be consistent with the permanent archive.
All systems participating in the control loop would remain at the Summit.
This is a change from the baseline, in which the ATArchiver moves to the Base.
There is no reduction in code development, however, because the AOS dataflow still requires a CCS DAQ Image Driver-triggered Butler ingest.
A disadvantage is that a transfer from the ATArchiver or its forwarder to the DBB at the Base must be implemented.

A second alternative would be to have this process be executed by the ATArchiver and an AuxTel OODS at the Base.
One advantage here is that collocation enables use of the high-performance, high-reliability database instance (`Oracle <oracle>`_) at the Base
This would cease to be an advantage if a high-reliability Oracle instance could be placed at the Summit.
Another is that it potentially enables a more efficient transfer to the DBB.
The disadvantage is that the control loop could be interrupted by a network outage; we have typically avoided mounting Base systems at the Summit because of this possibility.

.. _satisfying-camera-diagnostic-cluster:

Camera Diagnostic Cluster
-------------------------

The ComCam/LSSTCam Diagnostic Cluster and the AuxTel Diagnostic Cluster receive their images from their corresponding CCS DAQ drivers.
These images are ingested into a local Butler repository within the Diagnostic Cluster.
They can then be processed and visualized by Camera-provided code, which can in turn use the Data Butler and LSST Science Pipelines Libraries.
Once again, any master calibrations are copied by a separate process from the DBB and ingested into the Butler repository.


.. _commissioning-analysis:

Commissioning Analysis
======================

Commissioning analysis involves rapid-turnaround analysis of images.
Such analysis may also be combined with commands to Observatory systems via SAL.
The analysis and commands could be part of a well-defined, version-controlled observing script, or they could be part of an ad hoc notebook.
This leads to three subsidiary use cases:

* Scripted analysis plus control
* Notebook analysis plus control
* Analysis only

The first of these is handled in the same way as the `full-frame AOS processing <satisfying-active-optics>`_.
Images are sent via the ATArchiver, the ComCam Archiver, or the LSSTCam Archiver to the OODS.
Batch jobs triggered by SAL commands from the ScriptQueue are executed by the OCS-Controlled Batch service; these pipelines retrieve the OODS data via the Data Butler.
Results are returned in the command acknowledgement or published as telemetry.
If the results are large, they would be stored in the Engineering and Facility Database (EFD) Large File Annex (LFA).
OCS-Controlled Batch jobs generally run on the `Commissioning Cluster <comm-cluster>`_, but for AuxTel they could run on the Summit Analysis machine.

The second use case is handled by the Summit Analysis machine, which supports execution of notebooks and has access to the SAL-based control network.
This is expected to be used for AuxTel and ComCam, which produce images small enough to be analyzed on that machine.
Images are retrieved from the OODS (or the AuxTel Diagnostic Cluster).
Note that this OODS resides at the Base for ComCam.

LSSTCam is not expected to be able to use this mechanism in typical cases when the whole focal plane must be analyzed; instead, it would use the scripted mechanism above.
This is because placing the entire Commissioning Cluster on the SAL-based control network is risky and because providing sufficient compute for rapid full-focal-plane processing at the Summit is difficult due to power, cooling, and rack space limitations.
A possible alternative would be to support this use case via the Camera Diagnostic Cluster, which is already located at the Summit, but that would likely require substantial coordination with and development by the Camera software team that might pose difficulties.

The third use case is handled by notebooks running on the LSP instance in the Commissioning Cluster.
This instance will have a Portal Aspect to enable simple browsing of the available data from the OODS.
It will also have a Notebook Aspect to enable both ad hoc analysis and large-scale processing via batch jobs or Dask parallelization.

The timing of the availability of these services is given in `the following table <table-commissioning-timing>`_.

.. _table-commissioning-timing:

.. table:: Commissioning functionality by instrument and time.

    +------------+-------------------+--------------------------------------------------+
    | Instrument | Approx. Dates     | Functionality                                    |
    +============+===================+==================================================+
    | LATISS     |         — 2019-09 | * rsync from Tucson to LDF and Gen2 ingest       |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-09 — 2019-10 | * Single-host LSP with manual Butler ingest      |
    |            |                   | * rsync from Tucson to LDF and Gen2 ingest       |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-10 — 2020-11 | * In transit                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-11 — 2020-07 | * AuxTel Diagnostic Cluster and Summit Analysis  |
    |            |                   | * Minimal DBB from Summit to LDF and Gen3 ingest |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-07 —         | * AuxTel Diagnostic Cluster and Summit Analysis  |
    |            |                   | * Full DBB from Base to LDF and LDF to Base      |
    +------------+-------------------+--------------------------------------------------+
    | ComCam     | 2019-09 — 2019-11 | * Tucson OODS and single-host LSP                |
    |            |                   | * rsync from Tucson to LDF and Gen2 ingest       |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-11 — 2020-01 | * Tucson OODS and single-host LSP                |
    |            |                   | * Minimal DBB from Tucson to LDF and Gen3 ingest |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-01 — 2020-03 | * In transit                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-03 — 2020-07 | * Base OODS and Commissioning Cluster LSP        |
    |            |                   | * Base OODS and Summit Analysis                  |
    |            |                   | * Minimal DBB from Base to LDF and Gen3 ingest   |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-07 —         | * Base OODS and Commissioning Cluster LSP        |
    |            |                   | * Base OODS and Summit Analysis                  |
    |            |                   | * Base OODS and OCS-Controlled Batch             |
    |            |                   | * Full DBB from Base to LDF and LDF to Base      |
    +------------+-------------------+--------------------------------------------------+
    | LSSTCam    | 2021-03 —         | * Base OODS and Commissioning Cluster LSP        |
    |            |                   | * Base OODS and OCS-Controlled Batch             |
    |            |                   | * Full DBB from Base to LDF and LDF to Base      |
    +------------+-------------------+--------------------------------------------------+

.. note:: The LSP at NCSA is available at all times for analysis of DBB-conveyed images.

.. _latencies:

Latencies
=========

The Archivers are expected to transmit images to the OODS and the Data Backbone with a 2-second latency in normal operation.
The Data Backbone latency is expected to be low in normal operation, but it does more than the OODS in terms of file tracking, and it may experience outages or delays from time to time as it is dependent on more infrastructure services.
The OODS, on the other hand, is designed to be simpler and have higher uptime and lower latency, so that is the primary immediate-analysis pathway.
In particular, the "rsync" and "minimal DBB" mechanisms may take more than 15 minutes to begin the transfer of image data.
The "full DBB" mechanism will typically begin transfer of image data within seconds, but it is still considered an offline service subject to outages and delays.

The Camera CCS DAQ Image Driver code writes images with very low latency, but it does not include full headers as a result.

Butler ingestion is expected to be complete in a fraction of a second.

In the case of the Archiver interfaces to the OODS and DBB, hard links are expected to allow a single file to be shared between the two, minimizing latency.
In the event that files need to be written twice or copied, additional latency would be added.

.. _butler-repo:

Butler Repositories
===================

Image data is ingested into Butler repositories (initially Gen2, but soon Gen3) to enable standard usage by LSST Science Pipelines code.
Each Butler repository consists of a Datastore (in these cases, a simple Posix filesystem) and a Registry database.
For Gen3 repositories with the current Butler design, any code that needs to write output Butler datasets (which most if not all existing PipelineTasks do) must have write access to the same Registry database as the input datasets, although not necessarily to the same tables.
(Gen2 repositories only needed write access to the registry for ingestion or calibration ingestion tasks, not ordinary processing/analysis tasks.)
As an alternative to the current baseline, it may be possible to loosen this restriction in a future iteration of the Butler Registry implementation.

SQLite
------

SQLite Registries are used at the Summit on the Camera Diagnostic Cluster and potentially the AuxTel OODS if one is provided at the Summit.
Registry implementations in SQLite are appropriate only when there are a limited number of well-known readers and writers that can be trusted with full database access.
Because SQLite locking works on the entire database, large-scale queries need to be avoided, meaning that only `butler.get()` calls and PipelineTasks with fully-specified DataIDs should be used.

Oracle
------

Oracle Registries are used at the Base where a wide variety of users and usages must be supported.
As an alternative to the baseline, it may be possible to deploy Oracle at the Summit as well, adding flexibility at the cost of increased maintenance effort.


.. _summit-resources:

Summit Resources
================

A variety of computing environments are available on the Summit.

.. _tssw-machines:

TSSW Machines
-------------

Each CSC runs on its own (possibly virtual) machine or in its own container.
It is currently anticipated that the TSSW CSCs will be deployed and orchestrated via Kubernetes.

.. _sal:

These CSCs communicate via SAL, a pre-defined set of command, event, and telemettry messages passed over DDS.

.. _script-queue:

Script Queue
------------

Among the TSSW CSCs deployed on the Summit is the ScriptQueue, which allows Python scripts that send SAL commands and receive events and telemetry to be executed.
The ScriptQueue is the primary mechanism for automated control of the Observatory systems.

.. _ccs:

Camera Control System
---------------------

Multiple instances of the Camera Control System (including the ACCS for LATISS) run on Camera-dedicated hardware at the Summit.
The CCS is currently deployed via Puppet.

.. _ccs-daq-driver:

It has a component that retrieves images from the Camera Data Acquisition System and writes them to local files.
This CCS DAQ Image Driver can be extracted and deployed on other machines that have direct fiber links to the DAQ as necessary.

Each Camera Control System also has a Diagnostic Cluster (minimal for LATISS, larger for ComCam and LSSTCam) on a Camera-private network.
The Camera Diagnostic Cluster is designed to be used for automated rapid quality assessment of images and can be used to run an image visualization service.
For those uses, it is expected to provide low-latency ingestion of raw data into a Butler repository.
It is not currently designed for *ad hoc*, human-driven analysis.

.. _summit-analysis:

Summit Analysis Machine
-----------------------

A small system for human-driven analysis will be deployed on the Summit.
This system may initially be as small as a single node running Kubernetes and JupyterHub, intended to support the commissioning of the Auxiliary Telescope and LATISS.
Although this has yet to be demonstrated under Kubernetes, it should be possible for notebooks deployed on this system to send and receive SAL messages.
It will be possible to connect to this system remotely, through appropriate firewalls and/or VPNs.
Stringent security is required if it is allowed to issue SAL messages.
Any expansion of this system at the Summit is limited by the power, cooling, and rack space available in the Summit computer room, so we instead plan to expand its analysis capability by adding nodes at the Base in the Commissioning Cluster.

.. _summit-shared-filesystem:

Summit Shared Filesystem
------------------------

A modest-performance, modest-reliability shared filesystem is available on the Summit; its primary use is expected to be user home directories and not direct support of observatory systems.

.. _summit-artifact-repository:

Summit Artifact Repository
--------------------------

A repository for RPM, JAR, and Docker containers will be available at the Summit.


Base Systems
============

.. _archiver:

Archivers
---------

For the initial part of Commissioning of the Auxiliary Telescope, from mid-2019 to early-2020, the Auxiliary Telescope Archiver machine, currently in the Tucson lab, will be located at the Summit.
After that date, it will move to the Base.
The ATArchiver machine acquires images from LATISS, and a process on that machine arranges for them to be transferred to the Data Backbone, initially at NCSA but later at the Base.

For ComCam and LSSTCam, the Archiver machines reside at the Base.

.. _comm-cluster:

Commissioning Cluster
---------------------

Starting in early 2020, the Commissioning Cluster, a Kubernetes cluster at the Base, will provide an instance of the LSST Science Platform (LSP), including a portal, notebooks, visualization services, parallel compute (e.g. Dask), and batch computing services.
It will be able to access data from the AuxTel OODS (at the Summit or Base), the OODS at the Base associated with the ComCam/LSSTCam Archiver, as well as data from the Data Backbone.
The Commissioning Cluster will be equivalent to the current lsst-lsp-stable instance running in the production Kubernetes cluster at NCSA; its LSP code will be updated infrequently under change control, but its Science Pipelines containers can be updated much more frequently as needed.
It is not expected that the Commissioning Cluster will be able to communicate via SAL; it is solely for analysis and computation.
The Commissioning Cluster will be accessible remotely with appropriate security, similar to that for existing staff LSP deployments at NCSA.

.. _oods:

Observatory Operations Data Service
-----------------------------------

The Observatory Operations Data Service (OODS) will typically run on Archiver machines.
The OODS provides low-latency (seconds) ingestion of raw data into a Butler repository, and it manages that repository as a limited-lifetime cache.
The ATArchiver has its own internal filesystem that can be used for the OODS cache and can be mounted by other machines via NFS.
The OODS can also provide Butler ingestion of Engineering and Facility Database (EFD) Large File Annex (LFA) files, once those datasets and their ingestion have been defined.
The OODS cache will be the primary source of data for the Summit notebook-based analysis system.

The Summit systems can access data from the Data Backbone (DBB) at the Base, but they need to be prepared with fallback options if the network link is down or the DBB is down for maintenance.

.. _efd:

Engineering and Facilities Database
-----------------------------------

The full contents of the Engineering and Facilities Database are transported via Kafka messaging to NCSA for ingestion into the Data Backbone.
The Large File Annex is also ingested into the Data Backbone as Butler datasets and other files.
A short-term, time-series-oriented cache of most EFD contents optimized for analysis will be made available via an InfluxDB instance.

.. _dbb:

Data Backbone
-------------

The DBB, also available at the Base in early 2020, provides more-reliable but longer-latency ingestion of raw data and EFD LFA files than the OODS, and it keeps historical data as well as master calibration data products prepared by the Calibration Products pipelines.
The DBB, via the `Consolidated Database <oracle>`_, contains a transformed version of the EFD as a relational database.
Because raw data and the master calibrations that are needed to reduce it need to be in the same Butler, current master calibration data products will also be pushed to the OODS.

.. _oracle:

Consolidated Database
---------------------

The Base hosts an instance of the Consolidated Database, implemented as an Oracle RAC cluster, that contains tables for managing the DBB, the message content of the EFD, and the Registries for the OODS and other Butler repositories.
This instance is designed for high performance and high reliability, but individual schemas (such as the DBB schema) may be inaccessible for substantial downtime due to schema migrations or other maintenance activity.

.. _ocs-batch:

OCS-Controlled Batch
--------------------

The OCS-Controlled Batch CSC will provide access to batch analysis services, typically running on the Commissioning Cluster, via SAL commands that can be executed via the Script Queue CSC.
This allows automated analysis of images in the OODS to be performed in conjunction with other CSC commands.
Historical data from the DBB is also available, although through a separate Butler instance that is not integrated with the OODS instance.
Results of the batch job are returned in the command completion acknowledgement message or as separate telemetry (potentially including EFD LFA files).
The OCS-Controlled Batch CSC performs all translations to and from SAL messages; the batch service it uses therefore does not need to be on the SAL control network.

.. _prompt-processing:

Prompt Processing
-----------------

The Prompt Processing Ingest CSC at the Base obtains crosstalk-corrected images for ComCam and LSSTCam from the Camera (specifically the data acquisition system or DAQ) and transmits them to NCSA distributors which in turn make them available to automated processing pipelines.
These pipelines include the Alert Production and are expected to include prompt calibration quality control.
Results from these pipelines that are useful for Observatory operations are returned to the OCS through the Telemetry Gateway.
Other data products are transmitted via that Alert Distribution system and/or stored in the Data Backbone and made available through the LSP instances in the Data Access Centers, the Commissioning Cluster, or NCSA (for staff).

.. _chilean-dac:

Chilean Data Access Center
--------------------------

While the Base will host the Chilean Data Access Center (DAC) and an LSST Science Platform instance running on it, none of its facilities should be used for observatory operations as they are dedicated to serving science users.
Also, the Chilean DAC is not being built out until late in Commissioning.
To the extent that it is available and Commissioning or observatory staff has access to resources in it as scientists, it can be used for *ad hoc*, human-driven analysis.


NCSA Systems
============

NCSA hosts the general-purpose computing facilities for the project.
In Operations, these are primarily devoted to the Alert Production, Data Release Production, Calibration Products Production, and the US Data Access Center.
A substantial fraction is available through Commissioning and into Operations for staff use, including development, testing, quality assurance, and other uses.
This includes the staff instance of the LSP.

NCSA has access to all raw data, EFD data (in a Consolidated Database instance), and EFD LFA files, but the latency until it is available (via the Data Backbone), while typically short, may on occasion be longer due to outages or maintenance; NCSA systems that are not part of Prompt Processing are not required to have observing-level availability.

The Prompt data products (PVIs and difference images) are available where they are computed, at NCSA.
Access to them will be best handled by the NCSA LSP, although the DBB will also transfer them to the Base, where they are available to the Commissioning Cluster.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
