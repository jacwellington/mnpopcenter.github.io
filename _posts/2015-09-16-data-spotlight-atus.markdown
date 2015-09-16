---
title: 'Data Product Spotlight: American Time Use Survey'
teaser: 'Executive summaries for software developers  on MPC data'
author: ccd
categories: IPUMS
---


##    Introducing MPC Data Spotlights for Developers

In this series I'll briefly introduce  a number of MPC data products and highlight points of interest for software developers.  As a developer you don't necessarily need a deep understanding of a freely available dataset to put it to some interesting uses, but you do need to know what is in the data and what kinds of questions you can answer with it.

For every product I'll tell you 

1. What is  this thing in the first place? 
2. Specifics on the scope of the data and important details 
3. What (possibly non-obvious) interesting topics can I study / extract/ visualize  from the data? 
4. For MPC developers, what has been  unique and interesting about developing software to produce and distribute this particular product?

#### Getting the Data

All the MPC data products that offer microdata provide a data extraction system in which you specify  which datasets from within a product you want data and which variables / questions from the census or survey you want to study.  You can ask the system to produce a fixed-width ASCII file of the data, a CSV, or a binary version of the data for stats analysis programs such as SAS, Stata or SPSS. 

The files will  contain microdata, that is, one record per obvservation. Usually but not always these records represent people in the survey or census. In the cases of census data, records are only sampled from the census and anonimyzed; you cannot locate particular people in the census data.

For each of the products I'll give its address, and you can find instructions online on using the extract system.

#### Why Use MPC Data Products?

Even if you're not a social science researcher, using the MPC data products gives you a couple of powerful advantages: 

1. All data within a product (like U.S. Census data) has been made (by MPC) as comparable across the years as possible. This would be quite expensive to do on your own.
2.   With an extract of microdata you can produce new, never before created tables, and you can experiment with data to make tables that, within the limits of the possible,   best suit your needs.
3.  Systematic, consistent documentation of the data.


####  __ATUS__

__American Time Use Survey__

Find it at <a href="www.atusdata.org"> www.atusdata.org</a>.

-  **What is ATUS** 

The "American Time-Use Survey" tries to capture how Americans spend their time and with whom they spend it. ATUS has run annually since 2003. Some older U.S. time use data exists; other countries do time use surveys as well. The BLS is responsible for the ATUS. 

 - **Description of the Survey** 
 
 The "American Time-Use Survey" started in 2003. It has run annually since then. Participants in the survey are drawn from the Current Population Survey, a larger survey. Thirteen to twenty thousand households are included in the time-use survey and one individual from the household responds to a phone survey where they describe what they did with their time over a twenty-four hour period. The survey is spread throughout the year and different days of the week. 
 
 Participants provide starting and stopping times for all activities. The activities get categorized into some six hundred possible activities. Secondary activities get recorded as well with special  modules for eating and drinking and well being. Special attention is also paid to child care activities that might occur alongside other activities and care for elders. Participants are also asked who else was present during activities.
 
 - **Software Development Highlights**
 
 To make good use of the time use data one needs to summarize time spent doing activities, qualified by different things like time of day, where the activities were done, type of activity and so forth. The activity data is attached to each person and attached to each of these activity records are records concerning who was with the person... the data, in other words, is rather complex. It can't easily be flattened out, or rather that flat data would be  pretty hard to work with. 
 
 We offer a way for users to create their own time use variables which will get built at the time of a data extract and the values summarized and put onto each person record in the data  The data that's delivered to a user is person records with all the user created time use variables. The sorts of queries cannot be (quite) expressed in SQL. 
 
 Designing the web user interface to formulate the time use variable request, and keeping the UI concise but not confusing or misleading was an interesting challenge. The back-end process for creating these variables on the fly was also something not done in other products and and interesting difference from other microdata at MPC. It makes the data extract system more of a data creation system.
  
 
- **What can I do With ATUS** 

* Since ATUS participants  come from the CPS you might link the CPS people with ATUS people to get richer data on people doing various activities of interest. 
* You might do a limited time series since 2003 to observe, just for example, changes in computer use and television viewing.
* ATUS could provide useful data about sleep, which is hard to find. 
* You could look at time use differences between days of the week to assist with planning events or business hours.
* You could visualize various activities with time of day, day of the week or time of year as one of the axis and total time spent as the other.
* Since activities are listed in order for each person in the survey, you could add information on each activity record about the previous and subsequent activity. You could then answer questions like "What do people do after school?" or "How much time is spent driving before and after a restaurant meal?"
