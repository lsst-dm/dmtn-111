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
The Camera Diagnostic Cluster is designed to be used for automated rapid quality assessment of images and can be used for image visualization.
For those uses, it is expected to provide low-latency ingestion of raw data into a Butler repository.
It is not designed for *ad hoc*, human-driven analysis.

A modest-performance, modest-reliability shared filesystem is available on the Summit; its primary use is expected to be user home directories and not direct support of observatory systems.
A repository for RPM, JAR, and Docker containers will also be available at the Summit.

For the initial part of Commissioning of the Auxiliary Telescope, from mid-2019 to early-2020, the Auxiliary Telescope Archiver machine, currently in the Tucson lab, will be located at the Summit.
After that date, it will move to the Base.
The AT Archiver machine acquires images from LATISS, and a process on that machine arranges for them to be transferred to the Data Backbone, initially at NCSA but later at the Base.
The machine could also run an instance of the Observatory Operations Data Service (OODS).
The OODS provides low-latency ingestion of raw data into a Butler repository, and it manages that repository as a limited-lifetime cache.
The AT Archiver has its own internal filesystem that can be used for the OODS cache and can be mounted by other machines via NFS.
The OODS can also provide Butler ingestion of Engineering and Facility Database (EFD) Large File Annex (LFA) files, once those datasets and their ingestion has been defined.

The Summit systems can access data from the Data Backbone at the Base, but they need to be prepared with fallback options if the network link is down or the DBB is down for maintenance.

Base Systems
============

Starting in early 2020, the Commissioning Cluster, a Kubernetes cluster at the Base, will provide an instance of the LSST Science Platform (LSP), including a portal, notebooks, visualization services, and batch computing services.
It will be able to access data from the AuxTel OODS (at the Summit or Base), the OODS at the Base associated with the ComCam/LSSTCam Archiver, as well as data from the Data Backbone.
The DBB, also available at the Base in early 2020, provides more-reliable but longer-latency ingestion of raw data and EFD LFA files than the OODS, and it keeps historical data as well as master calibration data products prepared by the Calibration Products pipelines.
The DBB, via the Consolidated Database, contains a transformed version of the EFD as a relational database.
A short-term, time-series-oriented cache of most EFD contents optimized for analysis will be made available via an InfluxDB instance at the Base; the timing for its deployment is not yet known but is likely to also be early 2020.
Because raw data and the master calibrations that are needed to reduce it need to be in the same Butler, current master calibration data products will also be pushed to the OODS.
The Commissioning Cluster will be equivalent to the current lsst-lsp-stable instance running in the production Kubernetes cluster at NCSA; its LSP code will be updated infrequently under change control, but its Science Pipelines containers can be updated much more frequently as needed.

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


Satisfying the Use Cases
========================

CSCs on the Summit that use Science Pipelines code will retrieve pre-built containers from the Summit repository.
They will access data via the Butler from datastores on local filesystems, with SQLite registries (for Gen2 or Gen3) on the same filesystems.
Data can be retrieved from the DBB at the Base via a separate Butler (or a DBB-native API, when defined), but that data must be cached to the Summit filesystem for use when the Base is unavailable.
It is not anticipated that any Summit CSCs will need DBB data prior to the arrival of ComCam, so early 2020 is sufficient for installation of the Base DBB.
An Oracle database will not be deployed at the Summit.
In particular, no common central database will be provided for Butler registry or other Consolidated Database purposes; these will only be available at the Base.
If EFD data or LFA files are required, they should be obtained directly via SAL.

LATISS positioning and focus will run on a T&SS (possibly virtual) machine at the Summit using Butler access to a datastore NFS-mounted from the AuxTel Diagnostic Cluster, which will perform the Butler ingest into a local SQLite registry.
The Butler ingestion capability has not yet been tested in the Tucson lab, but it is required for the Camera's own processing of the images, and the code is similar to that of the OODS (although in Java rather than Python).
Similarly, Butler-ingested images on the (ComCam and LSSTCam) Camera Diagnostic Cluster will be used for Summit and Base visualization and Camera rapid automated analysis.

Full-frame wavefront processing and other Commissioning and calibration scripts will use the OCS-Controlled Batch service to execute their analyses as part of a Script Queue script.
Individual images may be quality-controlled by Prompt Processing if necessary.

For *ad hoc*, human-driven analysis, there are two time periods of note.
After early 2020, when the Commissioning Cluster and other Base facilities are available, the OODS at the Base and the Commissioning Cluster are the primary mechanisms, with the staff LSP instance at NCSA and the DACs as alternatives.
Between mid-2019 and early-2020, the AuxTel Archiver (and OODS) will reside at the Summit.
There are three alternatives during this period:

* Run notebooks within containers on the LATISS positioning/focus machine using the NFS mount from the AuxTel Diagnostic Cluster.
  While feasible, the LSP team prefers not to support notebooks running in this mode (outside the LSP environment).
* Run notebooks on a single-node LSP instance at the Summit.
  Such an instance would only be feasible at LATISS (single-CCD) scale.
  Configuring Kubernetes and the other required LSP services to run on a single machine may take a bit of work, but it can be useful for other reasons (such as enabling LSP testing).
  The Summit LSP would preferably use the datastore provided by the AuxTel OODS (including ingested EFD LFA files) rather than the AuxTel Diagnostic Cluster, as this will be most similar to Commissioning Cluster use of the Base OODS later on.
  For EFD data, it will be necessary to directly query the Summit EFD, as there is no alternative at the Summit or Base during this period.
* Run notebooks on the staff LSP instance at NCSA.
  Latency of access to raw data can perhaps be guaranteed to be faster during this time period.
  A quick test from the Summit showed that, with networking as of 2019-04-25, the demo ``Firefly.ipnb`` notebook could be run successfully including image display without undue interactive lag.
  EFD data can be retrieved from the Consolidated Database at NCSA or, if needed, from an InfluxDB replica.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
