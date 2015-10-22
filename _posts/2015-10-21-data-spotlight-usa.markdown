---
title: 'Data Product Spotlight:  IPUMS-USA'
teaser: 'Executive summaries for software developers  on MPC data'
author: ccd
categories: IPUMS
---


## MPC Data Product: IPUMS-USA (__Integrated Public Use Microdata Series__ for the U.S.A.)

Find it at <a href="http://usa.ipums.org"> usa.ipums.org</a>.

###  **What is IPUMS** 

The IPUMS is the flagship Minnesota Population Center data product. IPUMS-USA contains U.S. census microdata from 1850 to the present including supplemental datasets. The IPUMS began in 1990 with the first data release of fifty thousand sampled people from the 1880 U.S. census. Over the next ten years IPUMS datasets were integrated and released to cover census decades up to 1990. Since 2001 the U.S. Census also did an annual survey called the American Community Survey and we have included these surveys in the IPUMS data making it comparable with census data. We have also released integrated versions of the 2000 census data. Since 1990 new IPUMS samples have been added to increase the sampling density. Full count data, not samples have been released for 1880 and recently 1940. Eventually we plan to release the full census data for 1850 to 1940 and 1950 when the data is made public in 2022.


### **Description of IPUMS Data** 

IPUMS data is not a collection of statistics but rather individual records which IPUMS users can use to create their own statistics. Users can select particular questions of interest and select census years they want to study, then request an extract. For simple analysis there is an online tabulator.

IPUMS microdata covers the U.S. census decades from 1850 to 2010 and the annual American Community Survey from 2001 to the present.    In the past the census included a large number of questions to a subset of the population, known as the "Long Form." You can think of the ACS as something of a replacement for the long form questionaires -- the census no longer includes a long form. 

Prior to 1850 the census didn't record individual people but rather summarized each enumerated household. In future we plan to offer the household microdata from 1790 to 1840.

We don't offer the 1890 census, since the original forms burned long before they could be preserved. You can still find tabulations published from the original data, however.

Through a lot of effort MPC created samples of the  enumeration forms for the historical census data, making them machine readable. But this was only the first step. To  allow easy comparison of the United States population across time MPC created an "Integrated" form of the data, known as the Integrated Public Use Microdata Series.  

A lot of the value of IPUMS-USA is in the data quality, specifically the "integration" across census years. Integrating data consists of:

1. **Harmonizing responses to census questions** Similar or identical questions in different decades get the same numeric values for the same responses. Harmonizing also involves organizing the coding structure so that years where detailed questions were asked can be correctly compared with years where only general versions of those questions were asked. For instance some data has very detailed responses to questions of race while other decades only contain a few categories.
2. **Creating consistent variables across decades** For instance, occupation questions had different sets of valid responses in every decade, but it's possible to create a new occupation variable in the data which codes occupations the same for comparison purposes. This takes careful research to determine which occupations are actually comparable and the comparibility is not perfect, of course.
3. **Creating computed data that's comparable across time** For instance MPC produces the"pointer" variables which will indicate for each person who in the household is the subject's mother, father and spouse if those people are present. The pointers allow for attaching characteristics of relatives to other people and is a very useful addition to the data. Using the IPUMS you know the same method was used on all decades -- as much as possible -- to make the attached characteristics comparable.
4. **Imputing missing cases in the data** Since the census forms were filled out by hand a certain number were illegible and could not be entered. We use methods similar to the U.S. Census to impute values based on data from the rest of the population. If done carefully missing data imputation improves the data quality. All imputed cases get flagged in the data.

### **IPUMS-USA Software Challenges and Highlights**

As the first MPC data, IPUMS    had the oldest code base. Originally the data integration  software was written in Fortran. The first version of the data extraction system ran as a Perl-CGI site with a Fortran program to produce data extracts in the background. So, modernizing the architecture was the first challenge. Since then we've re-written in C++ for the integration application, Ruby for the website and Java for the back-end data extraction jobs.   For large scale tabulation and some large data processing and extraction work we're beginning to use Apache Spark on many MPC microdata products.

IPUMS-USA solves some additional unique problems through software not found in other MPC  data: Imputing small percentages of missing cases, and imputing -- essentially data mining -- values on all people in certain years.  

The 1850, 1860 and 1870 censuses  didn't ask people their relationship to the household head or their marital status. These turn out to be very powerful pieces of information, which, if known, allow for computing the "pointer" variables mentioned earlier and better imputation of missing data, as well as comparison to later decades. We take a set of donor people from the U.S. 1880 census and develop a model to predict their relationship and marital status values, then apply it to the earlier decades. The technique resembles a classifier but with a lot of tuning applied to get the best results.  Other data-mining techniques came close but couldn't quite match the accuracy of our method.

Imputing missing cases on many different variables is a simple problem in essence, but applying the solution across large datasets  correctly and efficiently is not so simple. In the days of the Fortran integration application RAM was at a premium and yet donor values had to reside in RAM if the program was to finish within a reasonable time (think days rather than weeks,) so special techniques had to be used to select the best donors only. It worked well but the result is a stateful process which cannot run in parallel. As we increased the size of our data releases we needed to parallelize our integration process, necessitating the development of a stateless missing data server. 

the Census Bureau  will continue conducting the census and ACS -- if all goes well -- into the indefinite future. The software must allow for building and execution  for decades. There's a balance to strike, of course, and we wouldn't expect to have no resources available to  modernize the code base, but it's a risk. We need the USA integration software to execute in a non-networked environment (secure research data centers) where external dependancies need to be minimized.

Finally, the IPUMS-USA data spans a large  time period. More effort is needed to integrate the data across time than with any other data product. Additionally the expectations for data quality are very high for the USA data. This requires research and special knowledge of U.S. historical  demography and the Census. Applying this knowledge results in the most complex domain specific code of any of the MPC products.

### **What can I Study With IPUMS?** 

##### __Family Structure and Individuals in the Context of their Families__

 The Census microdata allows one to examine family structure comparitively across time. This is a well studied  topic, but still interesting.  The data reveals  some facts that might be counter-intuitive to some. For example, you would find the  age of marriage at its lowest point around 1950, not in the 19th century.  See <a href="http://www.hist.umn.edu/~ruggles/Articles/Ruggles_Marriage_2014.pdf"> Marriage, Family Systems, and Economic Opportunity in the United States Since 1850. </a>

##### __Immigration and Language__

 The Census has questions about citezenship, birthplace, migration between states, race and language. You can study different regions to see how migration of different groups ebbed and flowed over the years.

##### __income, Housing and Mortgages__

Using ACS data You could study the rise and fall of home prices over the course of the housing boom and bust, just to measure it or to look for significant corelations.  

As homeowners lost their homes to foreclosure they moved into rental situations, reducing supply of rental housing and increasing rents. But the picture is rather nuanced. You could look not just at changes in numbers of houses / households rented vs. owned, but by using the microdata you could calculate total numbers of people living in rented housing vs. owned (by the household head) housing.   I might suggest that the demand for rental housing will  stay higher for longer than might be expected, because even as people leave larger rented households, they will continue to rent on their own due to loss of net worth and credit worthiness since 2008. 

There's data on second mortgages,house values, monthly rent and mortgage payments, HOA fees and utility costs in the ACS data. Linking that data with other demographic and housing data could yield interesting results.  

The ACS has enough records so that you can study different regions of the U.S. and individual cities like Phoenix or Minneapolis, not to mention the larger cities and metro areas.

Here's an interesting article that uses ACS and census data on housing costs and maps it: <a href="http://www.huduser.gov/portal/periodicals/cityscpe/vol15num3/Cityscape_Nov2013.pdf#page=257"> Exploring Housing Cost Data with Conditioned Choropleth Maps.” Cityscape: A Journal of Policy Development and Research 15: 251-256</a>.

##### __Historical and Current Economic Inequality__

 The U.S. Census  has  asked questions about income, occupation and education   over many decades. Rather than tracking economic progress in absolute terms you could compare one group of people to another by computing difference in incomes and types of occupations as a ratio. Specifically you could take any two groups: Teachers vs. farmers, white vs. black, urban vs. rural, and see if the ratio of economic difference varies over time or place with differences in government policy. These topics naturally are well studied, but  developing a framework for doing this kind of analysis on a variety of groups over time would be new and it would require IPUMS data.

##### New Data on Race and Same-Sex Partnership

In 2000 and 2010 the Census has allowed more than one response to the question of race and same-sex partnerships (marriage was not an option but an "unmarried partner" response was collected.) Using the microdata you can study households with much more nuance regarding race and same-sex partner relationships and families. 

----------

Be sure to check our <a href="bibliography.ipums.org"> bibliography</a> to discover many papers, books and articles using the IPUMS.
