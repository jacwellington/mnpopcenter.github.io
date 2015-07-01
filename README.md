# The MPC IT Blog

This is the MPC IT Blog repository.  This blog is managed using the Jekyll blogging platform and is published as a GitHub Pages site.  Github Pages is a web site creation tool that is integrated into Github.  It simply uses Jekyll under the hood to automatically rebuild a web site any time changes are committed to the master branch of a specially-named repository.  The advantage to this approach over other blogging platforms (e.g. Wordpress) is that the resulting site is purely static - there is no PHP or database to worry about.  It also allows the blog to be managed within the git toolset, which is already familiar to MPC IT staff.

The blog is managed like any other git/Github repository.  Blog editors manage the master branch.  Blog authors create new content on feature branches and then submit pull requests to the editors.  The author and editor will collaborate on the feature branch, and once the post is ready the editor will merge it into the master branch, triggering a site rebuild to publish the new content.

## Instructions for Blog Authors

### How to Write a Post

There are two methods for authoring a blog post.  

#### Method 1: Online via the Github Web Interface

The advantage of this method is that everything can be done on the Github web site.  

1. Open a browser to this repository.
1. Navigate into the _posts directory.
1. Click the + icon to create a new file in the _posts directory. Use the naming scheme "YYYY-MM-DD-title-of-my-post.markdown" for your new file.
1. Create your post. Follow the directions for [Blog Post Content Requirements](#content_reqs) below.
1. When done, create a new branch for this change and submit a pull request.

#### Method 2: Locally via a Clone of the Repository

The advantage of this method is that you can author from the command line and when offline.

1. Clone the master branch of this repository locally on your machine.
1. Create a new branch.
1. Navigate into the _posts directory.
1. Create a new file in the _posts directory. Use the naming scheme "YYYY-MM-DD-title-of-my-post.markdown" for your new file.
1. Create your post. Follow the directions for [Blog Post Content Requirements](#content_reqs) below.
1. When done, add your new file, commit your changes, and push the branch to Github.
1. Go to the Github web interface for the repository, and [create a pull request](https://help.github.com/articles/creating-a-pull-request/). 

#### <a name="content_reqs">Blog Post Content Requirements</a>

## Instructions for Blog Editors

### How to Test the Site Locally

