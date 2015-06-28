---
layout: page
status: publish
published: true
title: Migrating a Very Large Project from SVN to Git
author:
  display_name: Jayandra Pokharel
  login: jayandra
  email: jayandra@umn.edu
  url: ''
author_login: jayandra
author_email: jayandra@umn.edu
excerpt: 'IPUMS is one of the oldest projects at the MPC, having been in active development
  for decades. Its first commit in the repository dates back to 2003, when IPUMS was
  managed with CVS.  It then migrated to Subversion several years later.  In this
  post, we''ll take a look at what''s involved in migrating a large, complex project
  like IPUMS from Subversion to Git. '
wordpress_id: 535
wordpress_url: http://tech.popdata.org/?p=535
date: '2015-03-10 12:59:54 -0500'
date_gmt: '2015-03-10 17:59:54 -0500'
categories:
- IPUMS
- Featured
tags: []
comments: []
---
<p>IPUMS is one of the oldest projects at the MPC, having been in active development for decades. Its first commit dates back to 2003, when IPUMS was first added to a version control system (VCS), which happened to be Concurrent Versions System (CVS), a popular VCS of that era. Several years later the project was migrated to Subversion (SVN), an improvement in terms of functionality and ease of use. Recently, however, the flexibility and increasing popularity of Git, combined with the University of Minnesota&rsquo;s introduction of its own enterprise GitHub instance, made a compelling case for the migration of the IPUMS codebase from SVN to Git. Based on this information, the IPUMS team decided to proceed with the code migration.</p>
<p>The first tool of choice was &ldquo;git svn&rdquo;, which is built into Git and acts as a bidirectional bridge between SVN and Git. However, in digging further I learned about the <a href="https://rubygems.org/gems/svn2git">svn2git</a> gem, and ended up using it instead. The gem basically implements &ldquo;git svn&rdquo; under the hood, but additionally does the conversion of all SVN branches and tags to Git branches and tags. The fact that github suggests the use of svn2git in an <a href="https://help.github.com/articles/importing-from-subversion">article</a> regarding importing from svn to git also tempted me to use the gem.</p>
<p>Converting smaller SVN repositories to Git using the gem is quick and easy. However, in the case of IPUMS, the massive size of the existing SVN repository turned out to be a major challenge. The SVN repository for the IPUMS project was over 4GB in size and comprised 113 branches, 830 releases, and 30,943 commits. Running the conversion locally turned out to be more than my poor development box could handle: the process would often get killed due to memory issues. So, I had to run it on one of our compute machines, which had more physical memory than my local machine. I ran the svn2git conversion command using Unix <a href="https://www.gnu.org/software/screen/manual/screen.html">screen</a> tool, such that I could detach the screen session and reattach it anytime to peek into the progress made:</p>
<p><code>screen svn2git https://[<i>svn domain</i>]/repos/mpc/ipums/ --trunk trunk/ --tags releases/ --branches branches/<br />
</code></p>
<p>The conversion process was tough even for the powerful compute machines, often resulting in the process getting terminated. Whenever the process got terminated, I would resume it using the &ldquo;rebase&rdquo; command in svn2git such that the conversion would pick up where it left off.</p>
<p><code>cd <conversion_folder> &amp;&amp; svn2git --rebase<br />
</code></p>
<p>After babysitting the process for a number of days, the conversion finally completed. The converted git repository was 2.02 GB in size.</p>
<p>After completing the conversion from SVN to GIT, I performed a number of Git cleanup activities. As we were permanently moving from svn to git, the SVN metadata used by svn2git to keep track of SVN repository could be deleted.</p>
<p><code>cd <conversion_folder> &amp;&amp; rm -rf .git/svn<br />
</code></p>
<p>This decreased the size of our git repository from 2.02 GB to 1.27 GB.</p>
<p>The VCS at its root level had multiple projects in it. The IPUMS team opted to separate these projects into their own repositories such that each project contained only the commits specific to that project. For this purpose the &ldquo;git filter-branch&rdquo; tool with the &ldquo;subdirectory-filter&rdquo; option was used:</p>
<p><code>git filter-branch --subdirectory-filter [directory name] --tag-name-filter cat -- --all<br />
</code></p>
<p>The tool scanned through the whole repo and filtered out the commits that were related to the specified subdirectory. This filter operation was fast, and in no time the individual projects were separated to their own git repositories. Finally our capistrano deployment tasks were modified such that they would use the Git repository instead of SVN, and the conversion process was completed.</p>
<p>Some of the key points to be noted while converting a very large project from SVN to Git have been listed below:</p>
<ul>
<li>Depending upon project size, the svn to git conversion can be done on a local development machine or a dedicated remote server. When running in remote server, using "<em>screen</em>" to detach/reattach to your session is a must.</li>
<li>When the conversion gets interrupted, the process can be resumed by using the rebase functionality of svn2git.</li>
<li>In the case of one way conversion from SVN to GIT, the <em>.git/svn</em>&nbsp;subfolder can be deleted to reduce the size of converted git repository.</li>
<li>The "<em>git filter-branch</em>" command can be used to filter out commits that are related to a specific directory.</li><br />
</ul></p>
