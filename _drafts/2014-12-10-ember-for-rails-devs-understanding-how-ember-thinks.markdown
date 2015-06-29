---
layout: post
status: publish
published: true
title: 'Ember for Rails Devs: Understanding How Ember Thinks'
author:
  display_name: Jake Wellington
  login: jakew
  email: jakew@umn.edu
  url: ''
author_login: jakew
author_email: jakew@umn.edu
wordpress_id: 338
wordpress_url: http://tech.popdata.org/?p=338
date: '2014-12-10 12:47:16 -0600'
date_gmt: '2014-12-10 18:47:16 -0600'
categories:
- Ruby on Rails
- NHGIS
- Featured
tags: []
comments: []
---
<p>In my <a href="http://tech.popdata.org/take-it-and-run-a-tale-of-risk-failure-and-the-beginning-of-a-javascript-journey/" title="previous post">previous post</a>, I talked about our project's desperate need for a major Javascript refactoring, and how I got over the fear of failure in tackling this huge challenge. Now that I've set out on this task, all I had to do was figure out where to go next. Easier said than done with the wealth of Javascript tools and frameworks to choose from. After consulting with the rest of the team I decided I would take a run at Ember.js to see if that would bear fruit. The following is an overview of what I learned about how Ember works, tailored to someone with a background in Rails.</p>
<p>I like to think of Ember in terms of its flow control. Rails has a structured flow control too, and the two are easily compared. In Rails our flow looks like this:</p>
<p>Routes &rarr; Controllers &rarr; (Models &rarr; Controllers) &rarr; Views &rarr; (Helpers &rarr; Views)</p>
<p>Here's what it looks like in Ember:</p>
<p>Router.js &rarr; Route Class &rarr; (Models &rarr; Route Class) &rarr; Controllers &rarr; Views &rarr; Templates</p>
<p>There are a few noticeable differences between the two, and I went through some culture shock before I fully grasped how things worked in Ember. Coming from Rails and with a bit of background in AngularJS I had never really worked with an MV* framework which isn't MVC. In both Rails and Angular the controller was the one responsible for binding the data to the view (in Angular you have an object called $scope that both the controller and the view can see.) Rails works in a very similar way, giving access inside the view to any instance variables declared in the controller. This means that within the logic of the controller you are supposed to bind models, functions, and whatever else you want to access within the view.</p>
<p>In Ember, however, you actually bind your models inside your route classes. Then the controller can decorate the model with its own methods/bindings before passing it onto the view. It's an interesting design decision, because it doesn't leave you with a lot of reasons to have fat controllers. In fact, I didn't even use custom controllers at all in my initial proof of concept, which was to reimplement a small part of our NHGIS app workflow using Ember.</p>
<p>Lets dive a little deeper into the two parts of routing in Ember. The <code>Router.js</code> file is the master route declaration which looks a lot like Rails' <code>config/routes.rb</code> file. It declares paths that the user can navigate to in a nested format. Here's an example in Ember:</p>
<p>[code lang="javascript"]<br />
{{{javascript<br />
//router.js<br />
ExtractCreator.Router.map(function() {<br />
        	this.route('add', { path: '/' }, function () {<br />
                    	this.route('results', { path: '/' });<br />
                    	this.route('geog_level_filters');<br />
                    	this.route('year_filters');<br />
                    	this.route('topic_filters');<br />
                    	this.route('dataset_filters');<br />
        	});<br />
});}}}<br />
[/code]</p>
<p>Unlike in Rails, these declarations do not require REST verbs to be attached to them - they are only navigation paths for the user, not a server waiting for different types of requests.</p>
<p>And here's what a Route class file looks like:</p>
<p>[code lang="javascript"]<br />
{{{javascript<br />
//routes/add_route.js<br />
ExtractCreator.AddRoute = Ember.Route.extend({<br />
        	model: function () {<br />
                    	return this.store.find('result', 1);<br />
        	}<br />
});<br />
}}}<br />
[/code]</p>
<p>There's a few important things going on here. The first is how it is binding the <code>AddRoute</code> object to the <code>ExtractCreator</code> object. In Ember we create one global variable that represents the entire application - <code>ExtractCreator</code> in our example. The name <code>AddRoute</code> corresponds to the declaration of the route <code>add</code> from the <code>Router.js</code>. After extending Ember's default route, I bind the model by using EmberData's store object. EmberData is a library that is tightly coupled with Ember and provides model synchronization services to a server.</p>
<p>After the route class file, we have the controller which now can be used to add functions to the model which are route-specific. If you don't define a controller for a specific route, then a generic controller will be automatically created and used.</p>
<p>Next, we have views. Views are files where you place custom javascript actions for a specific template. If something needs to happen when the user makes an action which doesn't directly affect a model, then that code goes here. For example I implemented a jqgrid.js table within my Ember application as follows:</p>
<p>[code lang="javascript"]<br />
{{{javascript<br />
// add_view.js<br />
ExtractCreator.AddView = Ember.View.extend({<br />
        	didInsertElement : function(){<br />
                    	this._super();<br />
  	// jqgrid code goes here<br />
        	}<br />
});<br />
}}}<br />
[/code]</p>
<p>I'm using the supplied didInsertElementHook to run code after the specific template has loaded on the page. There are many other hooks available and you can write custom callable functions as well.</p>
<p>Lastly, there are templates which contain mustache templates that get transformed into html. Templates are loaded in the following hierarchy:</p>
<ul>
<li>application.handlebars</li>
<li>add.handlebars</li><br />
</ul><br />
The files would look like this:</p>
<p>[code lang="javascript"]<br />
{{{html<br />
<!-- application.handlebars !--></p>
<section id="subnav">
   Some things<br />
</section></p>
<section id="lower-section" class="clearfix">
        	{{outlet}}<br />
</section></p>
<p><!-- add.handlebars --></p>
<section id="side-nav">
<ul class="add-nav">
                    	{{#link-to 'add.results'}}
<li>Add Data</li>{{/link-to}}<br />
                    	{{#link-to 'add.geog_level_filters'}}
<li>Filter Geog Levels</li>{{/link-to}}<br />
                    	{{#link-to 'add.year_filters'}}
<li>Filter Years</li>{{/link-to}}<br />
                    	{{#link-to 'add.topic_filters'}}
<li>Topics</li>{{/link-to}}<br />
                    	{{#link-to 'add.dataset_filters'}}
<li>Datasets</li>{{/link-to}}<br />
        	</ul></p>
<aside class="filter-overview">
                    	{{#if model.has_results}}</p>
<h3>Filtered Data</h3><br />
                    	{{else}}</p>
<h3>Total Data</h3><br />
                    	{{/if}}</p>
<h4>Source Tables: {{model.source_tables_count}}</h4></p>
<h4>Time Series Tables: {{model.ts_tables_count}}</h4></p>
<h4>Boundary Files: {{model.boundary_files_count}}</h4><br />
        	</aside><br />
</section></p>
<section id="main-view" class="clearfix">
        	{{outlet}}<br />
</section></p>
<p>}}}<br />
[/code]</p>
<p>The <code>{{outlet}}</code> directive means yield any sub template into this area. The <code>add.handlebars</code> gets put into the <code>{{outlet}}</code> part of the application template, and any sub templates of <code>add</code> get put into the <code>{{outlet}}</code> part of <code>add</code>. Sub templates might come from the results route or any of the filter routes as seen in the router.js file. The <code>{{model.ts_tables_count}}</code> directive means print out the <code>ts_tables_count</code> property of the model attached to this specific route. If that model changes for any reason, that value will also change immediately, without having to re-render the entire template.</p>
<p>So that's a quick peek at my first adventure into Ember.js. I'm still not sure that it's the best solution for our project, but I learned how Ember and Rails "think differently" about application workflow. I'll continue to explore what Ember can offer, and blog more about the process in the future. If you've found this helpful, please let me know in the comments. Thanks for reading!</p>
