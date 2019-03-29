<pre>
/***************************************************************************
 *   Copyright (C) by ETHZ/SED                                             *
 *                                                                         *
 *   You can redistribute and/or modify this program under the             *
 *   terms of the SeisComP Public License.                                 *
 *                                                                         *
 *   This program is distributed in the hope that it will be useful,       *
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of        *
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the         *
 *   SeisComP Public License for more details.                             *
 *                                                                         *
 *   Developed by Luca Scarabello <luca.scarabello@sed.ethz.ch>                                         *
 ***************************************************************************/
</pre>
# Description

This module implements a Real Time Double-Difference Event Location method as described in the paper "Near-Real-Time Double-Difference Event Location Using Long-Term Seismic Archives, with Application to Northern California" by Felix Waldhauser.

This module can also be used to perform (non-real-time) full event catalog Double-Difference relocation, which is covered by another paper: "A Double-Difference Earthquake Location Algorithm: Method and Application to the Northern Hayward Fault, California" by Felix Waldhauser et al.

The actual Double-Difference inversion method is currently performed by the hypoDD software, which has to be installed in the system (currently tested on version 1.3 and 2.1b).

## Compile

In order to use this module the sources have to be compiled to an executable. Merge them into the Seiscomp3 sources and compile Seiscomp3 as usual.
<pre>
# merge rtdd-addons and Seiscomp3
git clone https://github.com/SeisComP3/seiscomp3.git sc3-src
cd sc3-src
git submodule add -f https://gitlab.seismo.ethz.ch/lucasca/rtdd-addons.git src/rtdd
</pre>
For compiling Seiscomp3, please refer to https://github.com/SeisComP3/seiscomp3#compiling



# Getting Started

## 1. Install hypoDD software

Go to https://www.ldeo.columbia.edu/~felixw/hypoDD.html and dowload hypoDD. Untar the sources, go to src folder, run make. Done! 


## 2. Define a background catalog for real time relocations

New origins will be relocated using a background or reference catalog of high quality locations. You might already have one or not.

If you already have a high quality catalog on your seiscomp database that can be used as background catalog, then you can specify it in scrtdd configuration as a file containing a list of origin id (or event id, in which case the preferred origin will be used). The file format must be a csv file in which there must be at least one column called seiscompId, from which the ids will be fetched by scrtdd.

***E.g. file myCatalog.csv***

```
seiscompId
event2019ducmfd
Origin/20181214107387.056851.253104
event2019sfamfd
Origin/20180053105627.031726.697885
Origin/20190121103332.075405.6234534
event2019dubnfr
Origin/20190223103327.031726.346363
```

In the more general case in which we don't already have a background catalog, we have to build one using scrtdd. First select the existing candidate events, which will be relocated to achieve the quality needed for a background catalog and then write the ids of those events/origins in a file usign the format of the example above (myCatalog.csv). Once that's done, we run the command:

```
scrtdd --dump-catalog myCatalog.csv
```

Or, if the events resides on a different machine we can use the -d option:

```
scrtdd --dump-catalog myCatalog.csv  -d  mysql://user:password@host/seiscompDbName
```

The above command will generate three files: event.csv, phase.csv and stations.csv. Those file are the alternative extended catalog format, useful when the catalog information is fully contained in those files instead of requiring the catalog to be present on a seiscomp database. Note: it is not compulsory to dump the catalog to perfomr a relcoation, we could also relocate a catalog that uses the siescompId file format, but sometimes it might be useul to dump the catalog to perform some manual editing before relocating it.

Now it's time to run scconfig, go to scrtdd configuration and create a profile that we use for relocating this catalog. In the profile we have to specify the generated files as source catalog, and then configure the various SCRTDD relocation settings. Once done, it's time to relocate this catalog with the command:

```
scrtdd --reloc-catalog profileName
```

scrtdd will relocated the catalog and will generate again the files event.csv phase.csv and stations.csv. At this point we should check the relocated evetns and see if we are happy with the results. If not, we change scrtdd settings and relocate the catalog again until we are satisfied with the locations. Now change scrtdd profile configuration and set the relocated catalog as background catalog.

We are now ready to perform real time relocation!


## 3. Real time single origin relocation

Real time relocation is done in two steps, each once has its own set of options in the configuration.

Step 1: location refinement. In this step scrtdd compute a preliminary relocation of the origin using only differential travel times computed from catalog phases (the ones present in seiscomp database).

Step 2: the refined location is used to perform a more precise relocation using both catalog phases information and differential travel time computed using cross correlation between phase waveforms.

After step2 the relocated origin is sent to the messaging system. If step2 fails, then the relocated origin from step1 is sent to the messaging system.

If both step1 and step2 fail, then a relocation is reattepted at a later time, accordingly to `delayTimes` option.

Note: when performing the catalog relocation ("scrtdd --reloc-catalog") it is done in a single step

To test the real time relocation we can either run playbacks or use two command line options which relocate existing origins:

```
  -O [ --origin-id ] arg                Reprocess the origin (or multiple comma-separated origins)
                                        and send a message
```

E.g. if we want to process an origin or event, we can run the following command and then check on scolv the relocated origin (the messaging system must be active):

```
scrtdd -O event2019dubnfr
```

Alternatively we can reprocess an XML file:


```
  --ep arg                              Event parameters XML file for offline 
                                        processing of contained origins (imply 
                                        test option). Ech origin will be 
                                        processed accordingly with the matching
                                        profile configuration
```

E.g.

```
scrtdd --ep event.xml
```


## 4. Troubleshooting

Check log file: ~/.seiscomp/log/scrtdd.log 

Alternatively, when running scrtdd from the command line use the following options:

```
# set log level to debug and log to the console (standard output) insted of log file
--verbosity=4 --console=1
```

A useful option we can find in scrtdd configuration is `keepWorkingFiles`, which prevent the deletion of scrtdd processing files from the working directory. In this way we can access the working folder and check input, output files used for running hypodd. Make sure to check the `*.out` files, which contain the console output of hypodd (sometimes we can find errors only in there, as they do not appear in hypodd.log file).

