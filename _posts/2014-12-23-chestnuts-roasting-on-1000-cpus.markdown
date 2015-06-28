---
layout: page
status: publish
published: true
title: Chestnuts Roasting on 1000 CPUs
author:
  display_name: Jesse Erdmann
  login: jerdmann
  email: jerdmann@umn.edu
  url: ''
author_login: jerdmann
author_email: jerdmann@umn.edu
wordpress_id: 368
wordpress_url: http://tech.popdata.org/?p=368
date: '2014-12-23 14:44:05 -0600'
date_gmt: '2014-12-23 20:44:05 -0600'
categories:
- Data Processing
- Featured
tags: []
comments: []
---
<p>Tis the holiday season and in Minnesota that means two things:</p>
<ol>
<li>It is a good time to reflect on the year.</li>
<li>It is cold outside.</li><br />
</ol><br />
One of the many things to love about the MPC is that we have lots of data. &nbsp;Literally a good chunk of the people that have ever lived are somewhere in our data. &nbsp;I haven't done the calculations to figure out exactly what the percentage is, however, if I WERE to want to do that calculation, we also have the hardware available to do so.</p>
<p>As a member of the <a href="http://www.research.umn.edu/">Office of Research</a> at the <a href="http://umn.edu">University of Minnesota</a> we get access to use the machines at the <a href="http://msi.umn.edu">Minnesota Supercomputing Institute</a>&nbsp;(MSI). &nbsp;When we find ourselves working on a problem that brings our poor laptop, or indeed our small Torque cluster, to its knees we can move that data over to MSI and bend a few thousand cores and terrabytes of RAM at the problem. &nbsp;Have a use for some GPUs, sure, why not?</p>
<p>In some situations a particular package you need might not be available and requires a privileged account to install, or at least to easily install it and all of its dependencies. &nbsp;Alas, MSI hasn't given us root on their very expensive machines supporting the entirety of the research community here at Minnesota. &nbsp;Luckily, we also have access to <a href="http://aws.amazon.com/">Amazon Web Services</a>&nbsp;(AWS) for just these situations! &nbsp;Any R&amp;D work can be tested out there before working with MSI or other partners to deploy the necessary tools in an appropriate environment.</p>
<p>None of this really addresses point number two, however. &nbsp;It still remains cold outside. &nbsp;Neither MSI nor AWS will let us into their data centers unsupervised with a bag of chestnuts ready for the roasting. &nbsp;While MPC does supply us with a generally cozy working environment, sometimes you might want a quick blast of heat to shake off that chill. &nbsp;Hmm... maybe I really do want to run that calculation on my laptop...</p>
