---
author: mpcit
title: Interpolation With Apache Spark
teaser: How we used spark to write a fast and scalable interpolation process.
categories: 
- Programming
---

# Overview of Spatial Integration
I recently used spark to run a batch job to create production data. I started knowing very little about spark, hadoop, or HPC in general. The following is an overview of the problem that I was trying to solve. Future blog posts will go more in depth on how I used spark to solve it.

##The Problem
I work on a webapp called NHGIS, which delivers modern and historical demographic data online. All the data for NHGIS is categorized by variable (e.g. population) and tied to a specific geography and year. We have a feature called time series which allows users to download multiple years for the same variable and same geography. This makes it very easy to see how different variables have changed over time for a given area, however it can be misleading when geographies keep the same name but change geographic footprint. To illustrate this problem I’ll give the following example.

Let’s suppose that Minneapolis had a population of 100,000 in the year 2000, and a population of 200,000 in the year 2010. Now let’s say that in the year 2007 Minneapolis annexed a large portion of St. Paul. Is the growth due to more people actually living in a geographic area, or is the growth only due to a border change and demographically things have stayed pretty much the same? In this case, it is difficult to tell. This type is what is called a nominally integrated time series, because it was created by connecting names of places rather than actual geographic locations. Nominally integrated time series are useful to have, but can sometimes have data which isn't always as clean as we'd like it to be.

What we plan to provide to our users is spatially integrated extracts, where the geographic footprint stays the same for the data over a period of time. The user will be able to pick a geographic time, such as 2010, and multiple data times, such as 2000 and 2010, and be able to compare them side by side using the 2010 geographic footprint. 

##The Solution
To create spatially integrated time series we start at the smallest unit of geography possible, blocks. Then we ask for a crosswalk (from highly learned researchers) that maps blocks from 2000 to blocks from 2010. A few sample lines (with a header row) would look like the following:
BLOCK2000, BLOCK2010, WEIGHT
BLOCK A, BLOCK A, .5
BLOCK A, BLOCK B, .5
BLOCK B, BLOCK B, 1
BLOCK C, BLOCK C, 1

The first column is an identifier for a block from the year 2000, the second column is for a block from 2010, and the third is the amount of the 2000 block that should be allocated to the 2010 block. For example the first line (passed the header row) allocates half of 2000-Block-A to 2010-Block-A, and the second allocates the other half of 2000-Block-A to 2010-Block-B. 

Then we take that crosswalk, and transform the second column into our target geography level. For example we could create a block to state crosswalk file by finding which states the 2010 blocks are in and sum up the rows that map from the same 2000-block to 2010-state. Assuming that 2010-Block-A and 2010-BLOCK-B are in Minnesota and 2010-Block-C are in Wisconsin we would get the following new crosswalk:
BLOCK2000, BLOCK2010, WEIGHT
BLOCK A, MINNESOTA, 1
BLOCK B, MINNESOTA, 1
BLOCK C, WISCONSIN, 1

After that we take the new crosswalk and apply the weights to the BLOCK2000 data. So our source file for the year 2000 could look like this:
BLOCKNAME, Variable1, Variable2
BLOCK A, 100, 20 
BLOCK B, 300, 0 
BLOCK C, 50, 10 

And our output would look like this.
STATE, Variable1, Variable2
MINNESOTA, 100, 20
MINNESOTA, 300, 0
WISCONSIN, 50, 10

Finally we would sum together lines that have the same STATE value.
STATE, Variable1, Variable2
MINNESOTA, 400, 20
WISCONSIN, 50, 10

This final file would be the new file we use for 2000 data using 2010 geography times.

In practice this entire process is agnostic to the source year/geography and the target year/geography, but the same steps apply regardless.

## Spark Overview
Before I dive into the code, I want to give a quick overview of Spark for those unfamiliar. Spark is a framework for cluster computing that keeps its computational data in memory for as long as possible (to reduce slow disk I/O). Its infrastructure is as follows:
• A master server. Spark comes with its own implementation of a master server, but you can also use Mesos or YARN.
• Multiple slave servers. 
• Multiple apis used to interact with the master server. The most used apis are for Scala, Java, and Python. In this example, I use the Python api.

## Implementation Overview
I solved this problem with three different spark scripts:
• Convert To Time Series
• Transform Crosswalks
• Transform Geotimes
The first two can be done in any order, but the final one takes the other two’s output as input to its process. 

### Conversion to Time Series
This script harmonizes the raw aggregate data variables. These raw variables are come directly from what was given to us by the Census Bureau. Harmonization is a process where a set of universal variable types are created, which represent the same thing regardless of time or dataset. We take the raw data given to us and select out the raw variables which correspond to our harmonized variables. Sometimes we have to select more than one raw variable and add them together to get a harmonized variable. The variables produced are called “Time Series Variables”, because they are now part of a time series (having been standardized across time).

#### The Spark Script
Here’s the main spark script that I submitted to the spark server:

```python
from pyspark import SparkContext, SparkConf, StorageLevel
from functools import partial
import sys, shutil, os
ROOT_DIR = '/pkg/ipums/istads/spark/interpolation/aggregate_to_time_series/'
MODULE_PATH = ROOT_DIR + "lib/"
sys.path.append(MODULE_PATH)
from transforms import Transforms
from config import Config

config = Config.create_config_hash(ROOT_DIR)
transforms = Transforms(config)

conf = SparkConf().set("spark.driver.memory", "4g").set("spark.driver.maxResultSize", "0").set("spark.executor.memory", "4g")
sc = SparkContext(conf=conf, pyFiles=[MODULE_PATH + "transforms.py"])
if os.path.exists(config["OUTPUT_FILE"]): shutil.rmtree(config["OUTPUT_FILE"])


source_lines = sc.textFile(config["SOURCE_FILE"])
source_lines.persist(storageLevel=StorageLevel(False, True, False, False, 1))
sorted_time_series_tuples = source_lines.map(transforms.source_line_to_time_series_tuple).sortBy(transforms.time_series_tuple_sortable_geog_tuple)
time_series_var_lines = sorted_time_series_tuples.map(transforms.time_series_tuples_to_string)
time_series_var_lines.saveAsTextFile(config["OUTPUT_FILE"])
``` 

And here is its associated small library of transformation functions:

```python
from functools import partial

class Transforms:
  """Helper class for main script."""
  def __init__(self, config):
    self.config = config

  def source_line_to_time_series_tuple(self, source_line):
    """Returns a tuple representing time series values given a source line.

    The returned tuple has two values.

    The first value is geog header information. 
    This information is exactly the same as in the geog header part of the source line.
    The second value is a list of time series values for that block.
    """
    time_series_var_composition_to_time_series_var = partial(self.__line_and_time_series_var_composition_to_time_series_var, source_line)
    time_series_var_values = map(time_series_var_composition_to_time_series_var, self.config["TIME_SERIES_VAR_COMPOSITIONS"])
    geog_information = self.__geog_info_from_source_line(source_line)
    return (geog_information, time_series_var_values)

  def time_series_tuples_to_string(self, time_series_tuple):
    """Returns a string representing a given a (geog_header, time_series_values) tuple."""
    geog_header = time_series_tuple[0]
    time_series_vars = time_series_tuple[1]
    formatted_vars = map(lambda ts_var: "%9d" % ts_var, time_series_vars) 
    return geog_header + ''.join(formatted_vars)

  def time_series_tuple_sortable_geog_tuple(self, time_series_tuple):
    """Returns a string of the state and county, tract, and block."""
    geog_header = time_series_tuple[0]
    state_and_county = geog_header[self.config["STATE_AND_COUNTY_START"]:self.config["STATE_AND_COUNTY_END"]]
    tract = geog_header[self.config["TRACT_START"]:self.config["TRACT_END"]]
    block = geog_header[self.config["BLOCK_START"]:self.config["BLOCK_END"]]
    return (state_and_county, tract, block)

  def __geog_info_from_source_line(self, source_line):
    return source_line[self.config["GEOG_HEADER_START"]:self.config["GEOG_HEADER_END"]]

  def __line_and_time_series_var_composition_to_time_series_var(self, source_line, time_series_var_composition):
    adv_values = [int(source_line[adv_range[0]:adv_range[1]]) for adv_range in time_series_var_composition]
    return reduce(lambda a,b: a+b, adv_values)
```

### Spark Script Walkthrough

The first 12 lines are just setting up the script and the configuration. The script has access to two different library files, one for config functions and one for transformation functions. They are titled “Config” and “Transforms”, respectively.  Lines 10 and 11 create the config and transforms objects. 

The next section sets up spark, by creating a conf object and passing in some memory parameters (which probably are too high for this particular script, but don’t make a difference on the machines that I’m running on.) It then passes in that conf to the spark server along with the location of the transforms script which spark will need to make sure all of its worker nodes have (it will replicate it and send it to them). Finally it removes any old output file so spark doesn’t later error out when trying to save to an already populated location. 

The final section is the meat of the script, where it opens the source “raw” data file, calls persist to make sure that it is loaded into the workers’ memory, and then starts running transformations. When spark opens the file it splits it up into chunks and spreads the chunks out to each one of its worker nodes, allowing them all to operate on the data at once. 

The first transformation it runs one line 20 is a map of all the lines in the file. This map will turn the line, which is represented as a string, into a tuple where the first value is the geographic location information and the second value is a list of the time series vars that it derived from that line. 

Next, the script sorts these tuples using a different function, that parses out specific sortable information based on the first value of the tuple.

Finally, the script turns this tuple back into a string which is then saved as a line in the new data file. 

Spark is lazy in that it will withhold execution until an operation which requires an answer to a previous one to run. The two points in this script where spark actually executes jobs are when sortBy is called and when saveAsTextFile is called. The previous maps are then prepended onto the job (it will do the map before the sort or before the save). This lazy evaluation allows for spark to have to read from disk as little as possible.
