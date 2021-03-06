# MIKE+ Pipe Sizing

A script for sizing the pipes in a MIKE+ database.

  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Demonstration model](#demonstration-model)
  - [Preparing MIKE+](#preparing-mike)
  - [Workflow](#workflow)
  - [Technical details](#technical-details)


## Introduction

The script pipesize.py adjusts the diameter of circular pipes in a MIKE+ SQLite database for collections systems. It reads the maximum discharge from the corresponding MIKE 1D result file (\*.res1d) and derives the required size according to the [Manning formula]( https://en.wikipedia.org/wiki/Manning_formula). Roughness and bed slope are properties of the pipe and taken from the database. The outcome is a network with more or less free flow conditions, or at least a water level slope parallel to the bed slope.

## Prerequisites

You need Python 3.8. Then have to install [MIKE IO 1D](https://github.com/DHI/mikeio1d) which will automatically download its dependencies. All other packages used in the script are part of the [Python 3.8 Standard Library](https://docs.python.org/3.8/library/).

The script has been tested with MIKE+ 2022, but it should work with earlier versions too.

## Demonstration model

The release package inlcudes a demonstration model with the initial result, before running the script.

## Preparing MIKE+

We need some preparations in the "Pipes and Canals" editor or in table msm_Link respectively.

The script relies on a user defined field **“usrOrigDiam”**.  This field serves two purposes:

  - It tells the script which pipes to alter, as the script only changes “Diameter” in pipes where “usrOrigDiam” has a value. The other pipes remain unchanged. Caution, if *all* "usrOrigDiam" are empty, the script will change *all* pipes (I may modify this behavior in the future as it looks a bit dangerous).
  - Once the process is finished you can use the field to compare the final diameter with the original diameter.

Perform the following two steps in Pipes and Canals editor:

1. Create the user defined field “usrOrigDiam” of type Double.
2. Select all pipes you want to include into the process and use the Field Calculator to copy the current diameter from “Diameter” to usrOrigDiam”.

The script assumes circular pipes. Look out for other cross section while populating “usrOrigDiam”. In rectangular or user defined cross sections changes in “Diameter” have no impact on hydraulics. Egg shape uses "Diameter" but has a smaller area as a circular pipe with same diameter and will be undersized. If you want to make use of the script with those cross sections, you have to replace the type with a circular one of the same hydraulic radius.

The script uses field **"Manning"**, which is the local "per pipe" value. Please populate this field, even if your HD simulation uses the material specific values.

The script uses field **Slope** from the database. Please press "Calculate" in order to refresh the various derived fields inlcuding 'Slope'. 


![calculate slope](./images/dimensionierung_calculate-slope.png)

## Workflow

Place script, MIKE+ \*.sqlite-database and the \*.res1d-file into the same directory. Ideally, you’ve configured MIKE+ so that the res1d is automatically saved into the database’s directory after each simulation run.

![working directory](./images/dimensionierung_verzeichnis2.png)

### 1. Run HD simulation in MIKE+

Run th HD simulation with the current diameters. Load the results and check the hydraulics in a longitudinal profile.

![profile v01](./images/dimensionierung_profil-v01.png)

If not already the case, copy the result file \*\_Network.res1d in the same as the MIKE+ sqlite-database.

### 2. Run the script

Open the command window and run the script:

```
C:\Users\tht\myDemo>pipesize.py
```
The script issues various messages:

```
c:\Users\tht\myDemo>python pipesize.py
Importing necessary libraries...
Defining functions...
Searching res1d and sqlite files...
Current directory: c:\Users\tht\myDemo
Number of res1d-files in directory: 2
Latest res1d-file will be used: s10_BaseDefault_Network_HD.res1d
Current MIKE+ database: mp22GettingStartedCS.sqlite
Creating list of links from res1d...
Preparing desired link results...
Connecting to database and retreiving msm_Link data...
All records in msm_Link: 9
Records in msm_Link with given usrOrigDiam: 7
Remaining records in msm_Link with value in column Manning: 7
ATTENTION: Method doesn' work with negative slope.
Dropping records with a negative slope: 1
Remaining records with a positive slope : 6
Joining msm_Link with Link results...
Computing new diameter...
Records with increased diameter: 6
Reducing dataframe and converting back to list of tuples...
Updating database...
6 records updated

c:\Users\tht\myDemo>
```

The last line tells you how many pipes were updated in the MIKE+ database. 



### 3. Run a new simulation and refresh visuals in MIKE+

Run a new HD simulation, which now uses the diamaters adapted by the script:

![active simulation](./images/dimensionierung_active-simulation.png)

Refresh the results in the results panel:

![refresh results](./images/dimensionierung_results-refresh.png)

Reload the longitudinal profile, to see both the new diameters and the corresponding results.

![profile v02](./images/dimensionierung_profil-v02.png)

In case you are looking at the "Pipes and Canals" table, apply and then remove some filter, in order to refresh the table content.

### Repeat steps 2. and 3. until satisfied

Increasing the pipe diameters will increase the peak discharges, especially if the previous setup experienced flooding. But in turn larger peak discharges will lead to even larger diameters! Therefore you may need a few iterations until the pipe diameters don't change any more.

![profile v05](./images/dimensionierung_profil-v05.png)

## Technical details

If you want to mimic steady state conditions, the rainfall intensity should be constant. But the script works with any rainfall time series! 

The script uses field "Manning" from the Pipes and Canals editor, which is the local "per pipe" value. Please populate this field, even if your HD simulation uses the material specific values.

The Manning formula doesn't work with zero or negative bed slope, hence the script doesn't change pipes with such values.

The script doesn't apply the Manning formula directly, but takes into account that only specifice pipe diameters are available on the market. Per default the diameters have a step size of 50 mm, meaning that the script will use 50, 100, 150 etc. Change parameter *step* at the top of the script if you prefer a different step size like 100 mm. You can't disable the stepping, instead use a very small value like 0.1.

There is no minimum or maximum diameter implemented yet.

The script doesn't check for a vertical cover above the pipe. In the worst case the pipe crown might rise over ground level.

If there are several res1d-files in the directory, the script will use the latest one. This is useful when progressing in several iterations like described above. If you creat both \*\_Surface_runoff.res1d and \*\_Network_HD.res1d at the same time, it will use the HD file (because in the aphabetical order it comes first). If for some reason the chosen res1d file is not a network HD file, the script will fail with some error message.

The script does not work if MIKE+ uses a PostGreSQL database instead of a SQLite database.




