:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

Introduction
============

This tech note describes the uses of Data Management (DM) Science Pipelines code in observatory operations, including the computing and storage systems available to it as Commissioning proceeds.

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

In addition, the Science Pipelines environment is used as the basis for Telescope and Site Software (T&SS) Commandable SAL Component (CSC) development.

Summit Resources
================

A variety of computing environments are available on the Summit.

Each CSC runs on its own (possibly virtual) machine; some have additional resources available to them.
In particular, the Camera has a Diagnostic Cluster (minimal for LATISS, larger for ComCam and LSSTCam) on a Camera-private network.
The Camera Diagnostic Cluster is designed to be used for automated rapid quality assessment of images and can be used to run an image visualization service.
For those uses, it is expected to provide low-latency ingestion of raw data into a Butler repository.
It is not designed for *ad hoc*, human-driven analysis.

A small system for human-driven analysis is expected to be deployed on the Summit.
This system may initially be as small as a single node running Kubernetes and JupyterHub, intended to support the commissioning of the Auxiliary Telescope and LATISS.
Although this has yet to be demonstrated under Kubernetes, it should be possible for notebooks deployed on this system to send and receive SAL messages.
It will be possible to connect to this system remotely, through appropriate firewalls and/or VPNs.
Stringent security is required if it is allowed to issue SAL messages.
Any expansion of this system at the Summit is limited by the power, cooling, and rack space available in the Summit computer room, so we instead plan to expand its capability by adding nodes at the Base in the Commissioning Cluster.

A modest-performance, modest-reliability shared filesystem is available on the Summit; its primary use is expected to be user home directories and not direct support of observatory systems.
A repository for RPM, JAR, and Docker containers will also be available at the Summit.

For the initial part of Commissioning of the Auxiliary Telescope, from mid-2019 to early-2020, the Auxiliary Telescope Archiver machine, currently in the Tucson lab, will be located at the Summit.
After that date, it will move to the Base.
The AT Archiver machine acquires images from LATISS, and a process on that machine arranges for them to be transferred to the Data Backbone, initially at NCSA but later at the Base.
The machine will run an instance of the Observatory Operations Data Service (OODS).
The OODS provides low-latency (seconds) ingestion of raw data into a Butler repository, and it manages that repository as a limited-lifetime cache.
The AT Archiver has its own internal filesystem that can be used for the OODS cache and can be mounted by other machines via NFS.
The OODS can also provide Butler ingestion of Engineering and Facility Database (EFD) Large File Annex (LFA) files, once those datasets and their ingestion have been defined.
The OODS cache will be the primary source of data for the Summit notebook-based analysis system.

The Summit systems can access data from the Data Backbone (DBB) at the Base, but they need to be prepared with fallback options if the network link is down or the DBB is down for maintenance.

Base Systems
============

Starting in early 2020, the Commissioning Cluster, a Kubernetes cluster at the Base, will provide an instance of the LSST Science Platform (LSP), including a portal, notebooks, visualization services, parallel compute (e.g. Dask), and batch computing services.
It will be able to access data from the AuxTel OODS (at the Summit or Base), the OODS at the Base associated with the ComCam/LSSTCam Archiver, as well as data from the Data Backbone.
The Commissioning Cluster will be equivalent to the current lsst-lsp-stable instance running in the production Kubernetes cluster at NCSA; its LSP code will be updated infrequently under change control, but its Science Pipelines containers can be updated much more frequently as needed.
It is not expected that the Commissioning Cluster will be able to communicate via SAL; it is solely for analysis and computation.
The Commissioning Cluster will be accessible remotely with appropriate security, similar to that for existing staff LSP deployments at NCSA.

The DBB, also available at the Base in early 2020, provides more-reliable but longer-latency ingestion of raw data and EFD LFA files than the OODS, and it keeps historical data as well as master calibration data products prepared by the Calibration Products pipelines.
The DBB, via the Consolidated Database, contains a transformed version of the EFD as a relational database.
A short-term, time-series-oriented cache of most EFD contents optimized for analysis will be made available via an InfluxDB instance at the Base; the timing for its deployment is not yet known but is likely to also be early 2020.
Because raw data and the master calibrations that are needed to reduce it need to be in the same Butler, current master calibration data products will also be pushed to the OODS.

The AT Archiver is expected to transmit images to the OODS and the Data Backbone with a 2-second latency in normal operation.
The Data Backbone latency is expected to be low in normal operation, but it does more than the OODS in terms of file tracking, and it may experience outages or delays from time to time as it is dependent on more infrastructure services.
The OODS, on the other hand, is designed to be simpler and have higher uptime and lower latency, so that is the primary immediate-analysis pathway.

The OCS-Controlled Batch CSC will provide access to batch analysis services, typically running on the Commissioning Cluster, via SAL commands that can be executed via the Script Queue CSC.
This allows automated analysis of images in the OODS to be performed in conjunction with other CSC commands.
Historical data from the DBB is also available, although through a separate Butler instance that is not integrated with the OODS instance.
Results are returned in the command completion acknowledgement message or as separate telemetry.

The Prompt Processing CSC at the Base obtains crosstalk-corrected images for ComCam and LSSTCam from the Camera (specifically the data acquisition system or DAQ) and transmits them to NCSA distributors which in turn make them available to automated processing pipelines.
These pipelines include the Alert Production and are expected to include prompt calibration quality control.
Results from these pipelines are returned to the OCS through the Telemetry Gateway.

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


Satisfying the Use Cases
========================

CSCs on the Summit that use Science Pipelines code will retrieve pre-built containers from the Summit repository.
They will access data such as calibration product datasets via the Butler from datastores on local filesystems, with SQLite registries (for Gen2 or Gen3) on the same filesystems.
The Summit is required to use SQLite registries because only those can be embedded within a filesystem with no server required.
Data can be retrieved from the DBB at the Base via a separate Butler (or a DBB-native API, when defined), but that data must be cached to the Summit filesystem for use when the Base is unavailable.
It is not anticipated that any Summit CSCs will need DBB data prior to the use of ComCam, so July 2020 is sufficient for installation of the Base DBB.
An Oracle database will not be deployed at the Summit due to support and management difficulties.
In particular, no common central database will be provided for Butler registry or other Consolidated Database purposes; these will only be available at the Base.
If EFD data or LFA files are required, they should be obtained directly via SAL.

LATISS positioning and focus will run on a T&SS (possibly virtual) machine at the Summit using Butler access to a datastore NFS-mounted from the AuxTel Diagnostic Cluster, which will perform the Butler ingest into a local SQLite registry.
The Butler ingestion capability has not yet been tested in the Tucson lab, but it is required for the Camera's own processing of the images, and the code is similar to that of the OODS (although in Java rather than Python).
Similarly, Butler-ingested images on the (ComCam and LSSTCam) Camera Diagnostic Cluster will be used for Summit and Base visualization and Camera rapid automated analysis.

Full-frame wavefront processing and other Commissioning and calibration scripts will initially run in notebooks at the Summit, using data from the Summit or Base OODS.
When productionized, they are expected to use the OCS-Controlled Batch service to execute their analyses as part of a Script Queue script.
Individual images may be quality-controlled by Prompt Processing if necessary.

.. _table-label:

.. table:: Analysis functionality by instrument and time.

    +------------+-------------------+--------------------------------------------------+
    | Instrument | Approx. Dates     | Functionality                                    |
    +============+===================+==================================================+
    | LATISS     |         — 2019-09 | * rsync from Tucson to LDF and Gen2 ingest       |
    |            |                   | * LSP at LDF                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-09 — 2019-10 | * Tucson OODS and single-host LSP                |
    |            |                   | * rsync from Tucson to LDF and Gen2 ingest       |
    |            |                   | * LSP at LDF                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-10 — 2020-12 | * In transit                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-12 — 2020-07 | * Summit OODS and single-host LSP                |
    |            |                   | * Minimal DBB from Summit to LDF and Gen3 ingest |
    |            |                   | * LSP at LDF                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-07 —         | * Base OODS and Commissioning Cluster LSP        |
    |            |                   | * Full DBB from Base to LDF and LDF to Base      |
    |            |                   | * LSP at LDF                                     |
    +------------+-------------------+--------------------------------------------------+
    | ComCam     | 2019-09 — 2019-11 | * Tucson OODS and single-host LSP                |
    |            |                   | * rsync from Tucson to LDF and Gen2 ingest       |
    |            |                   | * LSP at LDF                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2019-11 — 2020-01 | * Tucson OODS and single-host LSP                |
    |            |                   | * Minimal DBB from Tucson to LDF and Gen3 ingest |
    |            |                   | * LSP at LDF                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-01 — 2020-03 | * In transit                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-03 — 2020-07 | * Base OODS and Commissioning Cluster LSP        |
    |            |                   | * Minimal DBB from Base to LDF and Gen3 ingest   |
    |            |                   | * LSP at LDF                                     |
    |            +-------------------+--------------------------------------------------+
    |            | 2020-07 —         | * Base OODS and Commissioning Cluster LSP        |
    |            |                   | * Full DBB from Base to LDF and LDF to Base      |
    |            |                   | * LSP at LDF                                     |
    +------------+-------------------+--------------------------------------------------+
    | LSSTCam    | 2021-03 —         | * Base OODS and Commissioning Cluster LSP        |
    |            |                   | * Full DBB from Base to LDF and LDF to Base      |
    |            |                   | * LSP at LDF                                     |
    +------------+-------------------+--------------------------------------------------+

For *ad hoc*, human-driven analysis, there are two time periods of note.

Before early 2020, the Summit-based LSP and the staff LSP instance at NCSA are the primary mechanisms.
The usability of the NCSA-based LSP from the Summit was demonstrated by a quick test on 2019-04-25 in which the demo ``Firefly.ipnb`` notebook was run successfully including image display without undue interactive lag.
Data is provided at the Summit via the OODS and the EFD.
rsync or a minimal Data Backbone (including file tracking) are used to transfer files to the LDF for access there.
At the LDF, EFD data can be retrieved from the Consolidated Database or, if needed, from an InfluxDB replica.

After early 2020, when the Commissioning Cluster and other Base facilities are available, the Commissioning Cluster becomes the primary mechanism for analysis, with data provided by the Base OODS.
The staff LSP instance at NCSA remains available.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
