---
layout: page
status: publish
published: true
title: Harmonizing Data at the MPC
author:
  display_name: Colin Davis
  login: ccd
  email: ccd@umn.edu
  url: http://colinsblog.org/
author_login: ccd
author_email: ccd@umn.edu
author_url: http://colinsblog.org/
wordpress_id: 84
wordpress_url: http://tech.popdata.org/?p=84
date: '2014-09-23 13:41:29 -0500'
date_gmt: '2014-09-23 13:41:29 -0500'
categories:
- IPUMS
- Data Processing
- Featured
tags: []
comments: []
---
<p>Here at the MPC, we deal with lots of different kinds of source data. &nbsp;The Data Conversion Program (DCP) creates the MPC's harmonized version of microdata for&nbsp;many of the <a title="Data Products" href="http://tech.popdata.org/data-products/">MPC's data products</a>.</p>
<h3>What&rsquo;s Special About MPC Harmonized Microdata?</h3><br />
Until the MPC's <a href="http://www.ipums.org/">IPUMS</a> (Integrated Public Use Microdata Series) project came along, social science researchers needed to hire programmers or write code themselves to use more than one machine readable U.S. census sample at once. &nbsp;That's because the Census Bureau's mission is to take the best possible current snapshot of the U.S. population, not a backwards-compatible one. &nbsp;A lot changes in a decade - different social issues rise to the forefront, statistical methods improve, the size of the population grows... as a result, the Bureau asks different questions each census, or allows different answers to the same questions. &nbsp;As a result, censuses are not natively comparable through time.</p>
<p>Before IPUMS, research that spanned just&nbsp;two decades, let alone longer time series studies, were therefore expensive to conduct. Even when the work was done it wasn&rsquo;t necessarily reproducible by other researchers, since each researcher would develop their own methods. IPUMS dramatically lowered the cost to do time series research on the U.S. population and made it easier in general to use census microdata in research. &nbsp;How did we achieve that?</p>
<p>IPUMS did two things: (1) Produced comparable &ldquo;harmonized&rdquo; U.S. public use census data, and (2) Produced new machine readable samples of decades not available prior to the IPUMS project. &nbsp;For the first time, researchers could download versions of US censuses that were comparable to each other, and obtain that data in one consolidated dataset with associated documentation to describe how the data were harmonized with each other. &nbsp;This was a major breakthrough and the IPUMS project helped enable or lower the cost of a lot of new research.</p>
<p>Following the success of IPUMS, the Minnesota Population Center (first called the Historical Census Project) applied similar methods to international census data that was already machine readable. At nearly the same time, MPC took on the harmonization of Current Population Survey (CPS) and North Atlantic Population Project (NAPP) data. &nbsp;A few years later the other data projects were added. &nbsp;Throughout this time, the DCP continually evolved and expanded to be able to transform all of these source data sets into harmonized MPC versions.</p>
<p>Harmonized data has these advantages:</p>
<ul>
<li>Coding schemes for common variables are consistent in all datasets within a product&nbsp;(such as CPS.)</li>
<li>Many new common variables introduced to the datasets for added comparability across products.</li>
<li>New &ldquo;constructed&rdquo; variables were added to the data, such as head-of-household pointers.</li>
<li>Missing values due to historical data entry or transcription errors are given imputed values to improve data quality.</li>
<li>Data is checked for consistency problems and corrected when possible.</li>
<li>One set of documentation for all datasets in a product.</li>
<li>Weight variables to adjust results to match the whole population.</li><br />
</ul><br />
There's an enormous research effort at the MPC which studies each of these source datasets and determines how best to harmonize them. &nbsp;That topic alone could be the subject of many articles. &nbsp;However, this article will introduce the technical part of the problem - once the researchers figure out how to transform each source dataset into a harmonized form, how do we actually process all that data to produce the harmonized output?</p>
<p>Enter the DCP.</p>
<h3>What DCP Does</h3><br />
In a nutshell, the DCP takes an input dataset and produces a harmonized version of that dataset which has mapped all of the original variables into harmonized variables, which can then be compared to the same variables in other harmonized datasets. &nbsp;Let's take a quick tour of the various pieces that come together to allow the DCP to do this process.</p>
<p><strong>DCP takes two inputs:</strong></p>
<ol>
<li>Standardized format input data file.</li>
<li>Metadata describing the harmonized output format and coding scheme for the project and a description of the input format and coding scheme for the input data.</li><br />
</ol></p>
<h4>Input Data</h4><br />
The input data file must consist of blocks of data. Typically these blocks are households of people consisting of a household record followed by records for all the people in the household. This isn't always the case, though. &nbsp;For example, in a time use study, blocks could instead represent a single&nbsp;surveyed person followed by activities the&nbsp;person did that day. The main point is that the data can either be flat or strictly hierarchical, and each record must have a record type identifier.</p>
<p>Currently DCP expects the input data to be fixed width records, as this was historically the standard format for census data, though CSV would be trivial to use instead. Typically the input data is sorted geographically, but there&rsquo;s no requirement for this, it simply improves data quality. The record type identifier is conventionally the first character on each line in the file but this isn&rsquo;t part of the standard.</p>
<p>Here&rsquo;s a sample from our unit tests that has (H)ousehold, (P)erson, and (A)ctivity records in it.</p>
<pre><code>H0020<br />
P0351JONES       99<br />
A110801<br />
H0020<br />
P0351JONES       99<br />
P0351JONES       99<br />
P0351JONES       99<br />
A110801<br />
H0020<br />
P0351JONES       99<br />
A110801<br />
P0351JONES       99<br />
P0351JONES       99<br />
H0020<br />
H0020<br />
P0351JONES       99<br />
</code></pre><br />
Each line is a record with coded, fixed-with variables and their values. &nbsp;For instance, perhaps that "1" in the person record "</p>
<p><code>P0351JONES 99</code></p>
<p>means that they are a male.</p>
<p><code>Imagine this kind of data but with lines that are hundreds or thousands of characters long, and files that are millions of lines long.</code></p>
<h4>Metadata</h4><br />
Next, you need the "map" which tells the DCP how to transform input data into harmonized output data. &nbsp;The researchers produce a mountain of metadata to define&nbsp;this process. &nbsp;The metadata maps the input coding scheme to the harmonized output coding scheme for all variables where there&nbsp;is a simple mapping. &nbsp;In some cases, no simple mapping exists and we have to instead take a rules-driven approach which might, for instance, take into consideration values from other members of the household in order to determine this person's relationship to the head of the household.</p>
<p>The metadata contains the information necessary to extract the actual data from the input data (e.g. column start and width, since it&rsquo;s a fixed width file.) Metadata also has the corresponding column information for the harmonized data which is used to produce the harmonized output. The default file format that we produce is hierarchical fixed width ASCII 8-bit ISO-8859-1. The data conversion program can also output data as CSV, one file per record type. &nbsp;You might be wondering why we don't use a more "modern" storage format for all of this data. &nbsp;We're constantly investigating alternatives, but it turns out that for many of our operations, our flat file representation of the data is still very efficient.</p>
<p>Metadata is too bulky to reproduce here, but conceptually there&rsquo;s a many-to-many relationship between census variables (e.g. Occupation) and the datasets they can be found in (e.g. "1910 Census"). That joining is stored in one of our critical metadata objects, the &ldquo;translation table&rdquo;. &nbsp;The translation tables are one per variable, with information about how to recode the variable for each input dataset into its harmonized form. &nbsp;The master list of variables, variables.xml, is another form of metadata that the DCP consumes.</p>
<p>With&nbsp;the variables.xml metadata, the translation table for each variable, and the metadata describing the input and output file formats, DCP now has all the information it needs to produce harmonized data files given input files.</p>
<p><strong>DCP produces these outputs:</strong></p>
<ul>
<li>Harmonized data, typically one dataset produced per process, although DCP does have some parallel execution modes which can be enabled for larger datasets.</li>
<li>Files with frequency tabulations for selected variables. &nbsp;These are useful for quality checks and also for displaying to users on the website.</li>
<li>Optionally, CSV files and DDL metadata for loading into a database.</li>
<li>Log file listing non-fatal warnings and detailed error messages, as well as running time and performance statistics.</li><br />
</ul></p>
<h3>What DCP Is</h3><br />
The DCP is written in C for performance. &nbsp;It's highly modularized to allow it to consume many different record types. &nbsp;It also features a rules layer which allows our researchers to specify more complex transformations than can be expressed in a simple translation table. &nbsp;We use this rules engine to generate new variables like those "head-of-household" pointers, as well as other use cases. &nbsp;Over the years, the DCP has grown to be incredibly configurable, taking in many different kinds of inputs, producing many types of output, and instrumented to provide lots of statistics on its performance and processing status.</p>
<p>The DCP is one of the key pieces of software that enables the MPC to do what we do, and it continually evolves to meet our changing needs. &nbsp;In future blog posts, we'll highlight aspects of the DCP in more detail, to illustrate some of the complexity that has been captured in this code base over time.</p>
