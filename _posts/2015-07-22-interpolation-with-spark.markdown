---
author: mpcit
title: Interpolation With Apache Spark
teaser: How we used spark to write a fast and scalable interpolation process.
categories: 
- Programming
---

# Overview of Spatial Integration
I recently used spark to create a process that generates new production-ready data. I started knowing very little about spark, hadoop, or HPC in general. The following is an overview of the problem that I was trying to solve, and the implementation details of how I solved it.

## The Problem
I work on a webapp called [NHGIS](http://nhgis.org){:target="_blank"} , which delivers modern and historical demographic data online. All the data for NHGIS is categorized by variables (e.g. total male population) and tied to a specific geography and year. We have a feature called time series which allows users to download multiple years for the same variable and same geography. This makes it very easy to see how different variables have changed over time for a given area, however it can be misleading when geographies keep the same name but change geographic footprint. To illustrate this problem I'll give the following example.

Let's suppose that Minneapolis had a population of 100,000 in the year 2000, and a population of 200,000 in the year 2010. Now let's say that in the year 2007 Minneapolis annexed a large portion of St. Paul. How much of the growth from 2000 to 2010 is due to more people actually living in the same geographic area vs. due to the border change?In this case, it is difficult to tell. This type of time series data is called a "nominally integrated" time series, because it was created by connecting the data using the names of places rather than actual geographic locations. Nominally integrated time series are useful to have and easy to create, but don't always give the most useful comparison of data, depending on the research question.

What we plan to provide to our users with is spatially integrated time series, where the geographic footprint stays the same for the data over a period of time. The user will be able to pick a geographic time, such as 2010, and multiple data times, such as 2000 and 2010, and be able to compare them side by side using the 2010 geographic footprint. In the earlier example, if we were to use "spatially integrated" time series standardized to 2010 geographic footprints, the population for Minneapolis in the year 2000 would have been greater than 100,000 as it would have used the larger 2010 geographic footprint of Minneapolis to figure out the population of that area in both 2000 and 2010.

Creating these sorts of spatially integrated time series is more challenging than nominally integrated time series.  I'll now present our approach. 

## The Solution
To create spatially integrated time series we start at the smallest unit of geography possible, [blocks](https://en.wikipedia.org/wiki/Census_block){:target="_blank"}. Then we ask for a "crosswalk" file (which is given to us from highly learned researchers) that maps census blocks from 2000 to blocks from 2010. A few sample lines (with a header row) would look like the following:

BLOCK2000, BLOCK2010, WEIGHT
BLOCK A, BLOCK A, .5
BLOCK A, BLOCK B, .5
BLOCK B, BLOCK B, 1
BLOCK C, BLOCK C, 1

The first column is an identifier for a block from the year 2000, the second column is for a block from 2010, and the third is the amount of the 2000 block that should be allocated to the 2010 block. For example the first line (passed the header row) allocates half of 2000-Block-A to 2010-Block-A, and the second allocates the other half of 2000-Block-A to 2010-Block-B.

How we determine that .5 of the 2000 Block A should map to the 2010 Block A is a fascinating topic in itself which I won't be able to explore in this article.  There are different approaches, ranging from the simple (allocation by which percentage of 2000 Block A is overlapping the 2010 Block A) to the complex (taking into account many other sources of data about the block, such as land use patterns and land cover classifications within different areas of the block).  For the purposes of this article, we will simply note that the researchers who created this crosswalk file chose an appropriate mapping strategy after much research.

Then we take that crosswalk file and transform the second column into our target geography level (we aggregate upwards into larger geographies such as counties or states). For example if we want to create a block-to-state crosswalk file, we start by finding which states the 2010 blocks are in (this example assumes 2010 BLOCK A and 2010 BlOCK B from the above sample are in Minnesota and 2010 BLOCK C is in Wisconsin):

BLOCK2000, STATE2010, WEIGHT
BLOCK A, MINNESOTA, .5
BLOCK A, MINNESOTA, .5
BLOCK B, MINNESOTA, 1
BLOCK C, WISCONSIN, 1

and then collapse by summing up rows that map from the same 2000-block to the same 2010-state. We get the following 2000block-to-2010state crosswalk:

BLOCK2000, STATE2010, WEIGHT
BLOCK A, MINNESOTA, 1
BLOCK B, MINNESOTA, 1
BLOCK C, WISCONSIN, 1

XXX THIS NEXT PART NEEDS SOME WORK, TOO ABSTRACT, LETS TALK - FRAN XXX

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

The actual workflow is a bit more complicated as a I'll talk about it a below, but this is the overall process that we want to use in order to create the new data.


## Spark Overview
Before diving into the code, here is a quick overview of [Spark](https://spark.apache.org/) for those unfamiliar. Apache Spark is a framework for cluster computing that has an extensive API, and very fast computational speeds. Its infrastructure is as follows:

* A master server. Spark comes with its own implementation of a master server, but you can also use Mesos or YARN.
* Multiple slave servers. 
* Multiple apis used to interact with the master server. The most used apis are for Scala, Java, and Python. In this example, I use the Python api.

## Implementation Overview
I solved this problem with three different spark scripts:

* Convert To Time Series
* Transform Crosswalks
* Transform Geotimes

The first script takes regular aggregate data variables from the source geog year(e.g. "Total Population"), and picks out the ones to be used by the time series. The second script takes a crosswalk file that transforms blocks from a source geog year (such as 2000) to blocks from a target geog year (such as 2010) and changes it to target a different geog level in the target year (such as 2010 states instead of 2010 blocks). The final script takes the the time series variable file from the first scirpt and the newly created crosswalk file from the second script, and transforms the geotimes of the source variables (such as taking data with a geotime of blocks in the year 2000 and transforming it to states in 2010). The first two scripts can be done in any order, but the final one takes the other two's output as input to its process.

## Step 1 - Conversion to Time Series
This script harmonizes the raw aggregate data variables. These raw variables are come directly from what was given to us by the Census Bureau. Harmonization is a process where a set of universal variable types are created, which represent the same thing regardless of time or dataset. We take the data given to us and select out the variables which correspond to our harmonized variables. Sometimes we have to select more than one raw variable and add them together to get a harmonized variable. The variables produced are called "Time Series Variables", because they are now part of a time series (having been standardized across time).

Here is the main python script:

~~~ python
from pyspark import SparkContext, SparkConf, StorageLevel
from functools import partial
import sys, shutil, os
ROOT_DIR = '/myrootdir'
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
~~~

Below is an associated small library of transformation functions. This is just for reference, as the main script calls many of these methods or passes them onto spark. There is no need to walk line by line through this script.

~~~ python
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
~~~ 

### Conversion to Time Series Script Walkthrough 

This is a walkthrough of the first (main) script shown above.

The first 12 lines are setting up the script and the configuration. The script has access to two different library files, one for config functions and one for transformation functions. They are titled "Config" and "Transforms", respectively.  Lines 10 and 11 create the config and transforms objects. 

The next section, lines 13 - 15, sets up spark by creating a conf object and passing in some memory parameters. It then passes in that conf to the `SparkContext` object along with the location of the transforms script which spark will need to make sure all of its worker nodes have (it will replicate it and send it to them). The `SparkContext` object is the primary avenue of communication with the spark server. Finally it removes any old output files so spark doesn't later error out when trying to save to an already existing location. 

The final section is the meat of the script, where it opens the source "raw" data file, calls persist to make sure that it is loaded into the workers' memory, and then starts running transformations. When spark opens the file it splits it into chunks and spreads the chunks out to each one of its worker nodes, allowing them all to operate on the data at once. 

The first transformation it runs on line 20 is a map of all the lines in the file. This map is where the bulk of the work is being performed. The function `source_line_to_time_series_tuple` takes in a string that reperesents one line of the input data file. It then creates a function called `time_series_var_composition_to_time_series_var` that takes in a single time series var composition. A time series var composition is a list of tuples that looks something like this: `[(0, 10), (11,16)]` where each tuple in the list represents a range of positions in the line to read and parse for a single source value. The new function reads in and sums up these values from the given line, yielding a single harmonized time series value. After that function is created, it is then mapped over each time series var composition to get all the time series var values required. Lastly `source_line_to_time_series_tuple` returns a tuple where the first value is geographic information for the given line, and the second value is a list of these derived time series var values.  

Next the script sorts these tuples by the geographic information from the first values in each of the tuples. 

Finally, the script turns this tuple back into a string which is then saved as a line in the new data file. 

Spark is lazy in that it will withhold execution until an operation which requires an answer to a previous one to run. The two points in this script where spark actually executes jobs are when sortBy is called and when saveAsTextFile is called. The previous maps are then prepended onto the job (it will do the map before the sort or before the save). The file saved is then used in the later transform geographies step.

## Step 2 - Transformation of Crosswalks
A crosswalk is a csv file where each line represents a weighted transformation from a source geography to a target geography. This script takes as input a crosswalk file which lists the following: blocks from a source year, their block counterparts in the target year, and the weight from 0 to 1 of how much population from the source geography is expected to be within the footprint of the target geography (there is also a percent area measurement that I am going to ignore for the purposes of this post). It outputs a new crosswalk that converts the target blocks counterparts into a different geography level.   

Here is the main python script:

~~~ python
import pickle, sys, shutil, os
from pyspark import SparkContext, SparkConf, StorageLevel
ROOT_DIR = 'myrootdir'
MODULE_PATH = ROOT_DIR + "lib"
sys.path.append(MODULE_PATH)
from transforms import Transforms
from config import CrosswalkConfig

config = CrosswalkConfig.create_config_hash(ROOT_DIR)
cwalk_transforms = Transforms(config)

conf = SparkConf().set("spark.driver.memory", "4g").set("spark.driver.maxResultSize", "0").set("spark.executor.memory", "4g")
sc = SparkContext(conf=conf, pyFiles=[MODULE_PATH + "/transforms.py"])
if os.path.exists(config["OUTPUT_FILE"]): shutil.rmtree(config["OUTPUT_FILE"])

headers = sc.textFile(config["HEADER_FILE"])
header_dict = headers.map(cwalk_transforms.transform_header_key_value)

crosswalk = sc.textFile(config["CROSSWALK_FILE"])
crosswalk_dict = crosswalk.map(cwalk_transforms.transform_crosswalk_key_value)

header_crosswalk_groups = header_dict.cogroup(crosswalk_dict)
block_to_tract_crosswalk = header_crosswalk_groups.flatMap(cwalk_transforms.header_crosswalk_groups_to_lines).reduceByKey(cwalk_transforms.sum_crosswalk_values).sortByKey().map(cwalk_transforms.key_value_to_line)
block_to_tract_crosswalk.saveAsTextFile(config["OUTPUT_FILE"])
~~~ 

Here is its library of helper functions:

~~~ python
# Helper functions for the main spark croswalk transform script
from functools import partial

class Transforms:
  """A grouping of transformation functions dependent on a config hash.""" 

  def __init__(self, config):
    self.config = config
    #Create the geog_id_extraction function from the config info
    self.__create_target_geog_id_from_geog_header_line = self.__extract_geog_vars_function_from_ranges_and_length(config["TARGET_RANGES"], config["HEADER_FILE_WIDTH"]) 

  def transform_header_key_value(self, line):
    """Returns a tuple. The first value is the block id, the second value is the entire line."""
    return (self.__block_id_from_geog_header_line(line), line)

  def transform_crosswalk_key_value(self, line):
    """Returns a tuple. The first value is the block id, the second value is the entire line."""
    block2000, block2010, weight, percent_area = line.split(',')
    return (block2010, line)

  def header_crosswalk_groups_to_lines(self, header_crosswalk_group):
    """Transforms a single header_crosswalk_group to many tuples. 

    Each header_crosswalk_group is a tuple of two values.
    First value: The target block_id for the group.
    Second value: Another tuple of two values. 
      The first value is the target geog header line corresponding to the group's block id.
      The second value is a list of all the crosswalk lines whose target block is the group's block id.

    Each returned tuple is a weighted crosswalk line.
    """
    key = header_crosswalk_group[0]
    value = header_crosswalk_group[1]
    try:
      header_line = list(value[0])[0]
    except:
      raise Exception("Index Error: " + str(key))
    crosswalks_within_target = list(value[1])
    target2010 = self.__create_target_geog_id_from_geog_header_line(header_line)
    crosswalk_line_to_crosswalk_tuple = partial(self.__crosswalk_line_to_crosswalk_tuple, target2010)
    return map(crosswalk_line_to_crosswalk_tuple, crosswalks_within_target)

  def sum_crosswalk_values(self, crosswalk_values1, crosswalk_values2):
    """Sums a two tuples by adding together their first values and second values. Returns the summed tuple."""
    return (crosswalk_values1[0] + crosswalk_values2[0], crosswalk_values1[1] + crosswalk_values2[1])

  def key_value_to_line(self, crosswalk_tuple):
    """Returns a string representation from a crosswalk_tuple.

    The crosswalk_tuple should be organized as follows.
    First value: (block2000, target2010)
    Second value: (weight, percent_area)
    """
    block2000, target2010 = crosswalk_tuple[0]
    weight, percent_area = crosswalk_tuple[1]
    return "{},{},{},{:.10f}".format(block2000, target2010, weight, percent_area)

  def __extract_geog_vars_function_from_ranges_and_length(self, ranges, length):
    range_ends = [end for start,end in ranges]
    range_ends_with_endpoints = [0] + range_ends + [length]
    range_end_pairs = zip(range_ends_with_endpoints[:-1], range_ends_with_endpoints[1:])
    justification_lengths = [j-i for i,j in range_end_pairs]
    def extract_geog_vars_from_line(line):
      justified_geog_vars_array = [line[r[0]:r[1]].rjust(just) for r,just in zip(ranges, justification_lengths)]
      return ''.join(justified_geog_vars_array + [justification_lengths[-1] * " "])
    return extract_geog_vars_from_line

  def __block_id_from_geog_header_line(self, line):
    state_county = line[self.config["STATE_AND_COUNTY_START"]:self.config["STATE_AND_COUNTY_END"]]
    tract = line[self.config["TRACT_START"]:self.config["TRACT_END"]]
    block = line[self.config["BLOCK_START"]:self.config["BLOCK_END"]]
    return "F" + state_county + tract + block

  def __crosswalk_line_to_crosswalk_tuple(self, target2010, crosswalk_line,):
    block2000, block2010, weight, percent_area = crosswalk_line.split(',') 
    return ((block2000, target2010), (float(weight), float(percent_area)))
~~~ 

The first 14 lines are configurations as in the last script. Line 16 opens up a headers file. This headers file has the complete geographic information for all the target geographies listed in the crosswalk file. This is what we will use to look up what larger geography the smaller geography within the crosswalk file is part of. Lines 17 caches this file in memory across the nodes, and line 18 transforms each line into a key value pair, where the key is a unique id for a target block in the crosswalks file and the value is the entire line itself. This will allow us to match up lines from the headers file with their associated lines in the crosswalks file.

Lines 20-22 open up the crosswalk file and transform it into key value pairs as well. The keys here are also block ids and the values are the entire line. It is important to note that each key in the `crosswalk_dict` function has a matching key in the `header_dict` function.

Line 24 is where a lot of the work occurs. The cogroup function takes one RDD full of key values and joins it with another, grouping together each value that has the same key attributed to it. The result of that function, `header_crosswalk_groups`, is a list of the following types of objects: `[key, [lines from header file associated with the key], [lines from crosswalk file associated with the key]]`. In order to do this function spark passes around data between its workers so they each have all the needed data to continue operations on their hcunk of the `header_crosswalk_groups` object.

Line 25 takes the groups created in line 24 and turns them into tuples with `flatMap`. Each one of the lines outputted by the `flatMap` represents a transformed line from the input file. In a given header crosswalk group, there will be the same number of output tuples created from this `flatMap` as there were `[lines from crosswalk file associated with the key]` within that group. The tuples have the following structure: `([source_block, new_target_geography_id], [weight])`. 

Still on line 25, the code then takes each one of those tuples outputted by the `flatMap` and adds reduces them by grouping up tuples with the same `[source_block, new_target_geography_id]` and summing their weights. Now each pair of `source_block` to `new_target_geographies` has only one tuple associated with it. Finally these tuples are sorted, turned into strings, and saved out to a text file.

## Step 3 - Transformation of Geotimes

This is where the final step of the proces takes place. This script takes in the crosswalk file created in Step 2 and the time series variables file created in Step 1, and outputs a file whose geographies have been standardized to the target year. It does this by applying the weights found in the crosswalk file to the values found in the time series variable file, linking up each source block found in the crosswalk file with the corresponding block line in the time series variable file. Then it sums up the values of each line that had the same target geography, and saves the results in a text file.  

Main script:

~~~ python
from pyspark import SparkContext, SparkConf, StorageLevel
import sys, shutil, os
ROOT_DIR = 'mydir'
MODULE_PATH = ROOT_DIR + "lib"
sys.path.append(MODULE_PATH)
from transforms import Transforms
from config import Config

config = Config.create_config_hash(ROOT_DIR)
transforms = Transforms(config)

conf = SparkConf().set("spark.driver.memory", "1g").set("spark.driver.maxResultSize", "0").set("spark.executor.memory", "1g")
sc = SparkContext(conf=conf, pyFiles=[MODULE_PATH + "/transforms.py"])
if os.path.exists(config["OUTPUT_ESTIMATE_FILE"]): shutil.rmtree(config["OUTPUT_ESTIMATE_FILE"])

crosswalks = sc.textFile(config["CROSSWALK_FILE"])
crosswalks.persist(storageLevel=StorageLevel(False, True, False, False, 1))
crosswalk_dict = crosswalks.map(transforms.transform_crosswalk_key_value)

blocks = sc.textFile(config["BLOCK_FILE"])
blocks.persist(storageLevel=StorageLevel(False, True, False, False, 1))
block_dict = blocks.map(transforms.transform_block_key_value)

crosswalk_block_groups = crosswalk_dict.cogroup(block_dict)

target_2010_estimate_files = crosswalk_block_groups.flatMap(transforms.crosswalk_block_groups_to_estimate_lines).reduceByKey(transforms.sum_target_values).sortByKey().map(transforms.target_key_value_to_line)
target_2010_estimate_files.saveAsTextFile(config["OUTPUT_ESTIMATE_FILE"])
~~~ 


Helper library:

~~~ python
from functools import partial
import math

class Transforms:
  """A grouping of transformation functions dependent on a config hash.""" 

  def __init__(self, config):
    self.config = config

  def transform_crosswalk_key_value(self, crosswalk_line):
    """Returns a tuple given a crosswalk line in the form of: (BLOCK_ID, CROSSWALK_LINE)."""
    block2000, target2010, weight, percent_area = crosswalk_line.split(',')
    return (block2000, crosswalk_line)

  def transform_block_key_value(self, source_line):
    """Returns a tuple given a source line in the form of: (BLOCK_ID, SOURCE_LINE)."""
    block2000 = self._get_block_from_source(source_line)
    return (block2000, source_line)

  def crosswalk_block_groups_to_estimate_lines(self, crosswalk_block_group):
    """Returns many tuples given a crosswalk_block_group. See crosswalk_block_groups_to_lines"""
    return self.crosswalk_block_groups_to_lines(self._source_line_and_crosswalk_line_to_weighted_target_line, crosswalk_block_group)

  def crosswalk_block_groups_to_upper_bounds_lines(self, crosswalk_block_group):
    """Returns many tuples given a crosswalk_block_group. See crosswalk_block_groups_to_lines"""
    return self.crosswalk_block_groups_to_lines(self._source_line_and_crosswalk_line_to_upper_bounds_target_line, crosswalk_block_group)

  def crosswalk_block_groups_to_lower_bounds_lines(self, crosswalk_block_group):
    """Returns many tuples given a crosswalk_block_group. See crosswalk_block_groups_to_lines"""
    return self.crosswalk_block_groups_to_lines(self._source_line_and_crosswalk_line_to_lower_bounds_target_line, crosswalk_block_group)

  def crosswalk_block_groups_to_lines(self, source_line_and_crosswalk_line_to_target_line, crosswalk_block_group):
    """Returns many tuples given a crosswalk_block_group.

    The source_line_and_crosswalk_line_to_target_line arguments is a function which takes in
    a source line and a crosswalk line, and returns a target line. This function will either
    apply the crosswalk weights, compute the upper bounds, or compute the lower bounds.

    Each crosswalk_block_group is a tuple of two values.
    First value: The source block id for the group.
    Second value: Another tuple of two values.
      The first value is a list of all the crosswalk lines whose source block is the group's block id.
      The second value is the single source block line with the group's block id.

    Each returned tuple represents a crosswalk line having been applied to the source line using the
    source_line_and_crosswalk_line_to_target_line function.
    """
    key = crosswalk_block_group[0]
    value = crosswalk_block_group[1]
    crosswalk_lines = list(value[0])
    if len(list(value[1])) == 0: return [] # Only needed when using a subset of all the source blocks
    source_block_line = list(value[1])[0]
    crosswalk_line_to_weighted_target_line = partial(source_line_and_crosswalk_line_to_target_line,source_block_line)
    return map(crosswalk_line_to_weighted_target_line, crosswalk_lines)

  def sum_target_values(self, target_values1, target_values2):
    """Sums two tuples of the same length by adding together each of their values."""
    agg_data_values = []
    for index, agg_data_value in enumerate(target_values1):
        agg_data_values.append(agg_data_value + target_values2[index])
    return tuple(agg_data_values)

  def target_key_value_to_line(self, target_key_value):
    """Returns a string given a target key value.

    The given key is the target2010 geog header.
    The given value is a list of all the standardized vars for that target.
    """
    target2010_geog_header = target_key_value[0]
    standardized_vars = target_key_value[1]
    standardized_vars_as_strings = map(lambda v: "{:12.2f}".format(v).replace(".", ""), standardized_vars)
    return target2010_geog_header + ''.join(standardized_vars_as_strings)

  def _source_line_and_crosswalk_line_to_weighted_target_line(self, source_line, crosswalk_line):
    """Returns a tuple representing a weighted target given a source line and a crosswalk line."""
    block2000, target2010, str_weight, percent_area = crosswalk_line.split(',')
    weight = float(str_weight)
    time_series_vars = [source_line[start:end] for start,end in self.config["TIME_SERIES_VAR_RANGES"]]
    weighted_time_series_vars = [float(weight) * float(var) for var in time_series_vars]
    return (target2010, weighted_time_series_vars)

  def _source_line_and_crosswalk_line_to_upper_bounds_target_line(self, source_line, crosswalk_line):
    """Returns a tuple representing an upper bounds target given a source line and a crosswalk line."""
    block2000, target2010, str_weight, percent_area = crosswalk_line.split(',')
    weight = float(str_weight)
    time_series_vars = [source_line[start:end] for start,end in self.config["TIME_SERIES_VAR_RANGES"]]
    weighted_time_series_vars = [math.ceil(float(percent_area)) * float(var) for var in time_series_vars]
    return (target2010, weighted_time_series_vars)

  def _source_line_and_crosswalk_line_to_lower_bounds_target_line(self, source_line, crosswalk_line):
    """Returns a tuple representing a lower bounds target given a source line and a crosswalk line."""
    block2000, target2010, str_weight, percent_area = crosswalk_line.split(',')
    weight = float(str_weight)
    time_series_vars = [source_line[start:end] for start,end in self.config["TIME_SERIES_VAR_RANGES"]]
    weighted_time_series_vars = [math.floor(float(percent_area)) * float(var) for var in time_series_vars]
    return (target2010, weighted_time_series_vars)

  def _get_block_from_source(self, line):
    """Returns a block identification code given a source line.
    
    The line given sould be a block line from the source year.
    The returned code is in the following form: "F(STATE_CODE)(COUNTY_CODE)(BLOCK_CODE)".
    """
    state_county = line[self.config["HEADER_STATE_COUNTY_RANGE_START"]:self.config["HEADER_STATE_COUNTY_RANGE_END"]]
    tract = line[self.config["HEADER_TRACT_RANGE_START"]:self.config["HEADER_TRACT_RANGE_END"]]
    block = line[self.config["HEADER_BLOCK_RANGE_START"]:self.config["HEADER_BLOCK_RANGE_END"]]
    return "F" + state_county + tract + block
~~~

### Conversion of Geotimes Script Walkthrough 
The first 15 lines of the main script are just configurations as seen in the previous scripts. Lines 18 - 24 open up the crosswalk and time series block files, and then map then to key value pairs where each key is an id for the source geography (blocks) and the value is the entire line.

Line 24 then joins together those two sets of key value pairs (the `crosswalk_dict` and the `block_dict`) based off of their keys, giving a list of the following tuples `(common_key, ([list_of_crosswalk_lines_with_common_key], [list_of_block_file_lines_with_common_key]))`.  The `list_of_block_file_lines_with_common_key` should only have 1 value, because there should be only one line in the time series file per block.

Line 26 does a `flatMap` that maps each group in `crosswalk_block_groups` multiple objects, one for each item in `list_of_crosswalk_lines_with_common_key`, and each of these outputted objects is the a tuple in the following format `(target_geographic_level, ([list_of_weighted_data_values_from_single_block_line])`. Then it takes each one of these tuples and reduces it by grouping together each tuple that has the same `target_geographic_level`, and then summing up each of the data values for that specific key, outputting tuples in the following form `(target_geographic_level, ([list_of_summed_data_values_from_all_weighted_block_lines_related_to_target_geography_level])`. Finally, each one of these tuples is sorted by their geography level and then transformed into a string and saved to a file.

## Conclusion
You can see that each one of these scripts follows a similar format, where it reads in multiple files, joins them together on a specific key, performs a map, performs a reduce (if needed), sorts the output, and thne saves it to a text file. The main scripts are small and high level, leaving the details of the implementation either to a supporting library or to Spark. Some possible next steps for this process are: allowing for multiple target goegraphy levels to be done at one time, and possibly using data frames instead of RDD's for a lot of this computation. Feel free to comment if you have questions on the process, or general questions about spark and how it can be used.
