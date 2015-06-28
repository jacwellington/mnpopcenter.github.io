---
layout: page
status: publish
published: true
title: Importing Fixed Length Data Using Ruby (Part Two)
author:
  display_name: Colin Davis
  login: ccd
  email: ccd@umn.edu
  url: http://colinsblog.org/
author_login: ccd
author_email: ccd@umn.edu
author_url: http://colinsblog.org/
excerpt: Last time, I discussed the Fixed Length Record (FLR) format and showed how
  to use the 'hflr' Ruby gem to read in hierarchical data in FLR format and produce
  Ruby structs. In this post, I'll demonstrate how to combine 'hflr' with a simple
  importer class to load a database with the FLR data. Then I'll show you how to get
  some real-world FLR data to import. Finally, I'll discuss specific issues that come
  up when importing this data, and explore the IPUMS microdata just a bit.
wordpress_id: 667
wordpress_url: http://tech.popdata.org/?p=667
date: '2015-05-07 00:08:41 -0500'
date_gmt: '2015-05-07 05:08:41 -0500'
categories:
- IPUMS
- Data Processing
- Featured
tags: []
comments: []
---
<p><em>Read part one of this post <a title="Fixed Length Record Data" href="http://tech.popdata.org/fixed-length-record-data/">here</a>.</em></p>
<p>Last time, I discussed the Fixed Length Record (FLR) format and showed how to use the 'hflr' Ruby gem to read in hierarchical data in FLR format and produce Ruby structs.&nbsp;In this post I'll demonstrate how to combine 'hflr' with a simple importer class to load a database with the FLR data. <a id="more"></a><a id="more-667"></a>Then I'll show you how to get some real-world FLR data to import. Finally, I'll discuss specific issues that come up when importing this data, and explore the IPUMS microdata just a bit.</p>
<h4>Efficiently Import Data</h4></p>
<p>One could simply write code such as the following to import small numbers of records into a database:</p>
<p>[ruby]<br />
flr_file.each_record do |record|<br />
  customer = Customer.new<br />
  customer.name = record.name<br />
  customer.street = record.street<br />
  customer.zip = record.zip<br />
  customer.save<br />
end<br />
[/ruby]</p>
<p>However, performance will degrade unacceptably as the size of the import grows to a few thousand records.</p>
<p>What's needed is a way to avoid: (1) Instantiating all records as ActiveRecord objects, and (2) Issuing an insert statement to the database for each of those records. The 'activerecord-import' gem groups the inserts and, optionally, avoids instantiation into ActiveRecord objects altogether. In essence, activerecord-import allows you to pass in a large set of data as arrays or hashes or AR objects and do one insert for the lot of them. Doing this improves import time by around 50x. The speed-up will depend on the database type, hardware, and validations (they can be enabled or disabled.) Passing data to activerecord-import as arrays or hashes is substantially faster than passing ActiveRecord objects.</p>
<p>Using activerecord-import, Your code would look like</p>
<p>[ruby]<br />
records_to_import = []<br />
batch_size = 20_000 # arbitrary but reasonable batch size<br />
fields = [:name, :street, :zip]</p>
<p>flr_file.each_record do |record|<br />
  records_to_import << record.to_a<br />
  if records_to_import % batch_size == 0<br />
    Customer.import(records_to_import,fields)<br />
    records_to_import = []<br />
  end<br />
end</p>
<p># Import the remaining records % batch_size<br />
Customer.import(records_to_import,fields)<br />
[/ruby]</p>
<p>The 'import' method can take an array of arrays, and a 'fields' argument giving the order of fields in each inner array. Alternatively 'import' may take an array of hashes, removing the necessity for the 'fields' argument as the names of fields and their values are linked in the hashes.</p>
<p>You may instead pass in already instantiated Active Record models, but this will be slower than the first two options.</p>
<p>We can package up the buffering logic shown in the code snippets above, along with a way to manage the various options for passing data to 'import'. In the file 'importer.rb' you can find the full source. <a href="https://github.com/ccdavis/flrdata"> flrdata example code on github.com</a>.</p>
<h4>Example Data</h4></p>
<p>In the Git repository there's an 'input_data' directory with a sample data file 'usa_00001.dat'. This is a slice of a much larger data file created with the IPUMS-USA data extraction service. The dataset will let us study the housing boom, bust and recovery. The data comes from the American Community Survey, covering the years 2001 to 2013. We have both household records and person records in this dataset. We can create some interesting tables using just the household records, but some tabulations will require the people associated with those households.</p>
<p>The example tabulations that will follow later on were done with the full dataset. The slice of data provided in the example source is only for testing the import code.</p>
<p><em>[To learn more about the IPUMS and to recreate the full 3.3 GB dataset in our example, See <a href="http://usa.ipums.org"> usa.ipums.org</a>. You may browse the system and begin requesting data without signing in, but to get your data you must create a free account.]</em></p>
<p>The 'codebook.txt' and 'usa_0001.sps' files were created by the data extract system and were downloaded with the data file. The 'codebook.txt' is a human readable description of the data file, and the .sps file is for the SPSS statistics analysis software to read, to help you to use the data in SPSS. The extract system also produces files for SAS and Stata. For extracts with only one record type, the extraction service offers CSV and native binary formats of all three stats applications.</p>
<p>To figure out what to pass to the FLRFile class initializer, you'd look at the code book or possibly the SPSS file supplied with the dataset:</p>
<p>[sql]<br />
record type "H".<br />
data list /<br />
RECTYPE 1-1 (a)<br />
YEAR 2-5<br />
DATANUM 6-7<br />
SERIAL 8-15<br />
HHWT 16-25 (2)<br />
HHTYPE 26-26<br />
STATEICP 27-28<br />
METAREA 29-31<br />
METAREAD 32-35<br />
CITY 36-39<br />
CITYPOP 40-44<br />
GQ 45-45<br />
OWNERSHP 46-46<br />
OWNERSHPD 47-48<br />
MORTGAGE 49-49<br />
MORTGAG2 50-50<br />
ACREHOUS 51-51<br />
MORTAMT1 52-56<br />
MORTAMT2 57-60<br />
TAXINCL 61-61<br />
INSINCL 62-62<br />
PROPINSR 63-66<br />
OWNCOST 67-71<br />
RENT 72-75<br />
RENTGRS 76-79<br />
CONDOFEE 80-83<br />
HHINCOME 84-90<br />
VALUEH 91-97<br />
.<br />
record type "P".<br />
data list /<br />
RECTYPE 1-1 (a)<br />
YEAR 2-5<br />
DATANUM 6-7<br />
SERIAL 8-15<br />
PERNUM 16-19<br />
PERWT 20-29 (2)<br />
RELATE 30-31<br />
RELATED 32-35<br />
SEX 36-36<br />
AGE 37-39<br />
MARST 40-40<br />
RACE 41-41<br />
RACED 42-44<br />
HISPAN 45-45<br />
HISPAND 46-48<br />
BPL 49-51<br />
BPLD 52-56<br />
YRIMMIG 57-60<br />
SPEAKENG 61-61<br />
RACESING 62-62<br />
RACESINGD 63-64<br />
INCTOT 65-71<br />
INCINVST 72-77<br />
.<br />
end file type.<br />
[/sql]</p>
<p>This is fairly convenient, and if you regularly needed to import IPUMS data you might write a simple script to parse this sort of code book. Whatever the source of your FLR data you'll need a description with comparable information.</p>
<p>In our example, we're loading data into a database, so we need to define database tables with corresponding columns. See the 'schema.rb' for the translation of the above SPSS code into an Active Record migration script like one found in a Rails project. Again, you'd probably write a migration generator script if importing extracts was a frequent task.</p>
<p>As you can see from the 'schema.rb' migrations we're making two tables, corresponding to the two record types in the data file 'usa_00001.dat': Household and person. We'll use Rails conventions and call these tables 'households' and 'people'.</p>
<p>Since the data come with keys to link the two tables together, we'll make sure to index the two columns holding those keys. The 'households' table has a column 'serial' which will get the value of the SERIAL variable -- it's a unique household serial number. Within one dataset SERIAL is unique; since we're extracting data across thirteen datasets, we'll need to use 'ACSYR' (the year) to differentiate records. The 'people' table has a 'serialp' column that will hold the SERIALP variable; SERIALP has the value of the household serial number for the household the person record belongs to. Again, we'd need the ACSYR (which is on the people table as well) to differentiate between datasets within the 'people' table. The tables already have the Rails standard 'id' columns as unique keys. A nice feature for the importer would be to add a 'household_id' column to the people table and copy the value of 'id' from the correct household record, so we wouldn't need to use composite keys in our joins. (I've done this in production code but it's rather specialized and gets in the way of the example code.)</p>
<p>To read the data we use the 'hflr' gem as in the above example code and in the previous post. We need to provide a layout to 'hflr'. To see the HFLR layout data, look in 'extract_layout.rb'.</p>
<p>To run the example, just do</p>
<p>[code light="true"]<br />
# ruby import.rb<br />
[/code]</p>
<p>You may use JRuby instead. The required gems are given at the top of the 'import.rb' file.</p>
<p>The import process should take a few seconds on the data included with the example code. Before importing data, the script executes the migrations in the 'schema.rb' file, rebuilding the database schema from scratch. The full dataset referenced in the 'codebook.txt' file has approximately thirty-one million person records and fourteen million household records. It will take between one and five hours to import depending on your storage hardware, CPU and Ruby version.</p>
<p>Once the import has finished we should do a few simple queries to verify that all records got imported. To make effective use of this data you'd need to read documentation at <a href="http://usa.ipums.org">usa.ipums.org</a>. You can begin exploring the data just with the variable category labels included in the codebook, however. For now, you just need to know that each household record has a weight variable and each person record has a person weight variable. The value of these variables, in the 'households.HHWT' and 'people.PERWT' columns respectively, indicates the number of people or households the record represents from the larger population, as this dataset is only a sample of the entire United States population. Divide the weight values by 100 (the two least significant digits are fractional amounts) to get a useful whole number version of the weight. For instance, the value 10100 really means "this record represents 101.00 records like it."</p>
<p>So, to see how many people the U.S. Census estimated were in the country in 2013, you would do:</p>
<p>[sql]<br />
sqlite> select sum(PERWT/100) from people where acsyr =2013;<br />
316128839<br />
sqlite><br />
[/sql]</p>
<p>And we can check the population growth:</p>
<p>[sql]<br />
sqlite> select sum(perwt/100), acsyr from people group by acsyr;<br />
277075792|2001<br />
280717370|2002<br />
283051379|2003<br />
285674993|2004<br />
288398819|2005<br />
299398485|2006<br />
301621159|2007<br />
304059728|2008<br />
307006556|2009<br />
309349689|2010<br />
311591919|2011<br />
313914040|2012<br />
316128839|2013<br />
sqlite><br />
[/sql]</p>
<p>From this result we know that not only were the right number of person records imported but they were records with correct weights, indicating that the data in those records is probably all correct. Of course we'd have to check every column to be absolutely sure. As a final check we can join the people with their household data to get geography and housing information for the state of Arizona (STATEICP=61) for every year in the data. The OWNERSHP variable will distinguish between people living in rented households (2) and owner occupied (1). OWNERSHP=0 is for other living situations like prisons, military bases, dormitories, etc.</p>
<p>[sql]<br />
sqlite> select people.acsyr,ownershp,sum(perwt/100)<br />
>from households,people<br />
>where people.serialp=households.serial and stateicp=61 and people.acsyr=households.acsyr<br />
>group by people.acsyr,ownershp;<br />
2001|1|3564953<br />
2001|2|1618471<br />
2002|1|3724464<br />
2002|2|1625033<br />
2003|1|3851262<br />
2003|2|1638885<br />
2004|1|3948678<br />
2004|2|1689667<br />
2005|1|4043947<br />
2005|2|1762319<br />
2006|0|109501<br />
2006|1|4185780<br />
2006|2|1871037<br />
2007|0|109370<br />
2007|1|4305747<br />
2007|2|1923638<br />
2008|0|119162<br />
2008|1|4327201<br />
2008|2|2053817<br />
2009|0|119010<br />
2009|1|4317435<br />
2009|2|2159333<br />
2010|0|139384<br />
2010|1|4086287<br />
2010|2|2188066<br />
2011|0|143926<br />
2011|1|4028111<br />
2011|2|2310468<br />
2012|0|149267<br />
2012|1|3991930<br />
2012|2|2412058<br />
2013|0|147624<br />
2013|1|4006706<br />
2013|2|2472294<br />
[/sql]</p>
<p>This all looks quite believable. It's interesting to notice how the ratio of people living in rentals to those living in owner-occupied households changed over the boom and bust cycle of the housing market. There are a few more tables included in the example source repository that explore the topic further.</p>
<h4>More Efficient Analysis</h4></p>
<p>The example pushes Sqlite3 to its limits; if you recreated my extract in full you'd need to wait an hour or two just to import it. Some of the queries require five or more minutes to run, even though the necessary indexes are present. (This shows why analyzing hierarchical data in a relational database can be challenging.)</p>
<p>The first option, clearly, is to use the same import script with a full fledged database server on the back-end. MySql or Postgres will not import much more quickly, but the queries will execute faster in many cases, especially if your server has a lot of memory and it's tuned correctly. I've provided a 'pivot()' and 'pivoted_html_table()' function because Sqlite3 doesn't support pivoting result sets. You won't need those helpers with a full featured database.</p>
<p>If you want to run frequent ad-hoc queries on this data, you might find that even MySql, Postgres or other databases are painfully slow, especially when joining the two tables. Column-store databases will allow for very fast tabulations across large tables when the tabulation involves limited numbers of columns. You could use a script similar to the example to load a column oriented database such as InfiniDB. Check <a href="https://mariadb.com/services/infinidb-services"> InfiniDB at Maria DB </a> to download it. Or try <a href="http://www.infobright.org/"> InfoBright</a>.</p>
<p>There's a plug-in for Postgres which may not perform quite as well as the above two options but integrates with a standard Postgres install and is super convenient. See <a href="https://www.citusdata.com/blog/76-postgresql-columnar-store-for-analytics">Citus Data</a> for a quick introduction and installation instructions.</p>
<h4>Alternatives to Databases</h4></p>
<p>Consider looking into the IPython / IRuby projects. IPython provides tools to analyze data locally and has visualization tools. IRuby is built on the same libraries as IPython but lets you script your work in Ruby.</p>
<p>Get the IRuby project source and documentation from <a href="https://github.com/minad/iruby"> IRuby on github.com</a>. You can also install IRuby with</p>
<p>[code light="true"]<br />
# gem install iruby<br />
[/code]</p>
<p>You should look into the JuPytR project as well (Julia, Python,Ruby). There are now many other languages supported as well as these three. Get it on the <a href="http://ipython.org/"> IPython page</a>.</p>
<p>When using IRuby or JuPyter you'd find the HFLR gem useful in formatting your hierarchical fixed-length data into sets of CSV files for easy consumption. You may run into size limitations, but not with even the full sized example extract in my example code.</p>
<p>Have fun.</p>
