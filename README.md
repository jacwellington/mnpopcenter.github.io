# The MPC IT Blog

This is the MPC IT Blog repository.  This blog is managed using the Jekyll blogging platform and is published as a GitHub Pages site.  Github Pages is a web site creation tool that is integrated into Github.  It simply uses Jekyll under the hood to automatically rebuild a web site any time changes are committed to the master branch of a specially-named repository.  The site is available at <https://github.umn.edu/pages/mpc/> (this will be aliased to tech.popdata.org if we go live with this approach). 

The advantage to this approach over other blogging platforms (e.g. Wordpress) is that the resulting site is purely static - there is no PHP or database to worry about.  And since it's hosted at Github, there is no web hosting service to worry about, either.  It also allows the publishing pathway to be based on git, which is already familiar to MPC IT staff.

The blog is managed like any other git/Github repository.  Blog editors manage the master branch.  Blog authors create new content on feature branches and then submit pull requests to the editors.  The author and editor will collaborate on the feature branch, and once the post is ready the editor will merge it into the master branch, triggering a site rebuild to publish the new content.

## Instructions for Blog Authors

### <a name="content_reqs">Creating Blog Post Content</a>

Jekyll posts are mostly just text files.  You can write your content in the editor of your choice.

By incorporating special syntax into your posts, you have access to a good deal of Jekyll functionality.  Each blog post file should start with front matter (described in the next section).  That signals Jekyll to run this file gets run through the Jekyll engine, which means that at site generation time, the file gets processed by an input converter (e.g. Markdown -> HTML or Textile -> HTML) and also by the template engine, which is called Liquid.  Liquid tags allow you to do things like invoke the syntax highlighting code block feature or retrieve the site URL value.

For this site we have standardized on markdown as our input format.  Specifically, [Kramdown markdown](http://kramdown.gettalong.org/syntax.html).

#### Blog Post Metadata - Jekyll front-matter

All Jekyll blog posts need to start with a YAML section called the "front matter".  This is the metadata for your post.  Front matter is delineated by three dashes "---".  Here is an example block of front matter:

~~~
---
title: 'Our IT Hiring Process: How and Why'
teaser: 'An overview of our IT hiring process and why we designed it this way.' 
author: fran
categories:
- Team
tags:
- hiring
- staff
---
~~~

Most front matter variables have site-wide defaults, so you don't need to worry about setting them in each post.  The minimum set of front matter fields you should include in each post are title, author and categories.  A teaser is strongly encouraged, and tags are optional but also encouraged.  There are other variables which can be used to change the layout of a post or otherwise alter behavior, but the site editors would generally add these if needed.  

For the `categories` variable, the set of allowable values is XXX coming soon XXX.

#### Blog Post Content - Kramdown Markdown

After the end of the front matter, the rest of the file contains your blog post content.  Use [Kramdown markdown](http://kramdown.gettalong.org/syntax.html) syntax to write your post.  Most, but not all, of the Kramdown functionality will work on Github Pages (this is mostly relevant for code blocks, read below on how to do this correctly).  All the usual stuff works, such as lists, headers, links, inline images, tables, and so on.

#### Including Code in Your Posts

Jekyll on GitHub Pages uses Pygments to provide syntax-highlighted code blocks in posts.  To use Pygments, use the tags `{% highlight _lang_ %}` and `{% endhighlight %}` to surround your code.  For example:

~~~
{% highlight ruby %}
flr_file.each_record do |record|
  customer = Customer.new
  customer.name = record.name
  customer.street = record.street
  customer.zip = record.zip
  customer.save
end
{% endhighlight %}
~~~

(I can't show you what this would produce on the blog because Github doesn't apply Jekyll to the page you're currently reading, so it doesn't recognize Liquid tags).

Pygments offers hundreds of languages.  Some useful ones for us are ruby, python, python3, perl, sql, bash, java, css, yaml, c, c++, js, json, html and text.

_Warning: Do not try to use other flavors of markdown fenced code blocks (e.g. "~~~" or "```").  They will not work in the Jekyll/GitHub Pages ecosystem._

To create inline code snippets, use markdown code spans with the single backtick.  These _do_ work on GitHub Pages sites.  For example:

~~~
At the shell, type `gem install hflr` to install the hierarchical fixed length records gem.
~~~

produces

At the shell, type `gem install hflr` to install the hierarchical fixed length records gem.

#### Including Images in Your Posts

If you want to include images in your posts, add them to the images directory in the repository.  Then, to reference them in your post, use this combination of markdown syntax and Liquid tag:

`![Image Alt Text]({{ site.url }}/images/filename.png)`

### Authoring Workflow 

There are two basic workflow approaches for authoring blog posts, each with its own advantages.

#### Method 1: Online via the Github Web Interface

**One thing I'm not sure about is Github permissions and branching vs. forking.  Since I'm a Github owner, it's hard for me to simulate this as a non-owner.  Someone please test this with branching and forking and see what works.**

* Pros: Everything can be done from the web.
* Cons: No way to preview the post.
* Best for: Very simple posts. Posts written externally that you just want to cut and paste. Fixes to existing posts. 

1. Open a browser to this repository.
1. Navigate into the _posts directory.
1. Click the + icon to create a new file in the _posts directory. Use the naming scheme "YYYY-MM-DD-title-of-my-post.markdown" for your new file.
1. Create your post. Follow the directions for [Creating Blog Post Content](#content_reqs) below.
1. When done, submit a pull request back to the main repo.

####  Method 2: Locally by Cloning the Repo and Running Jekyll Locally

**One thing I'm not sure about is Github permissions and branching vs. forking.  Since I'm a Github owner, it's hard for me to simulate this as a non-owner.  Someone please test this with branching and forking and see what works.**

* Pros: Edit on local machine with your preferred tools.  Ability to preview locally (with Jekyll properly installed locally).
* Cons: Additional one-time setup required. Still have to go to Github to create pull request.
* Best for: Authoring more complicated posts. Collaborative editing with others.  

_Note: If you don't have Jekyll installed, please see [Instructions for Installing and Running Jekyll Locally](#install) first._

1. Clone the master branch of this repository locally on your machine.
1. Create a new branch with a descriptive name like "fran-hiring-process-post".
1. Navigate into the _posts directory.
1. Create a new file in the _posts directory. Use the naming scheme "YYYY-MM-DD-title-of-my-post.markdown" for your new file.
1. Create your post. Follow the directions for [Creating Blog Post Content](#content_reqs) below.
1. Start your Jekyll server if you haven't already, and preview your work.
1. When done, add your new file, commit your changes, and push the branch to Github.
1. Go to the Github web interface for the repository, and [create a pull request](https://help.github.com/articles/creating-a-pull-request/). 

## <a name="install">Instructions for Installing and Running Jekyll Locally</a>

1. Clone the repository to your local machine.
1. (Optional) Create a new gemset in your Ruby environment.
1. From the root of the repository, run `bundle install`.

Now you have Jekyll and its dependencies installed.  To start the Jekyll server:

1. Run the command `jekyll serve --config _config.yml,_config_dev.yml`
1. The site should be viewable at http://localhost:4000

Note that Jekyll will report that it started on http://0.0.0.0:4000/, but at least on my system the server is not reachable at this address, you have to use http://localhost:4000/.  

You can keep the server running, and it will automatically detect changes to files in the repository and rebuild the site accordingly.  At this point you can iterate over your blog post authoring using Method 2 above, and preview the site locally as you go.