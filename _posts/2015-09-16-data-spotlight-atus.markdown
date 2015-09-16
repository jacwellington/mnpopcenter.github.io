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

