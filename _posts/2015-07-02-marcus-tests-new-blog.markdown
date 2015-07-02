---
title: 'Marcus Tests the New Blogging Platform'
teaser: 'Marcus is stuck between a rock and another rock. Will the new blogging platform save him?' 
author: marcus
teaser: This post will be awesome, read it!
categories:
- Team
tags:
- testing
- blog
- rock
---

Rock - Marcus - Another Rock
============================
Blah blah blah. Some smart programming discussion, other good stuff.
We have every expectation of continuing the project beyond 2012, but will have to secure further funding as our current grants expire. To be successful, we need to have a large body of users and published works we can point to. Please inform us if you have any presentations or publications using IPUMS data.

Well, Let's See Some Code, Dude
===============================

Are you sure you want to `sudo rm -rf /`? Okay, it's your funeral, buddy.
	
{% highlight ruby %}
class API::Metadata::V1::FrequenciesController < ApplicationController
  def index
    begin
      variable_name = params[:variable]
      sample_name = params[:sample]

      unless variable_name || sample_name
        raise 'Variables and/or samples must be included in request'
      end

      frequencies = Frequency.by_variable_and_sample(variable_name, sample_name)
      render json: frequencies, status: :ok
   rescue => e
      render json: e.message, status: :bad_request
    end
  end
end
{% endhighlight %}

How About an Image?
===================

Gonna be Star Wars, isn't it? Always with the Star Wars.

![Star Wars is good for you!]({{ site.url }}/images/sw.jpg)
