---
layout: post
title:  "Making Craft sing with Varnish and nginx"
date:   2015-06-05 15:30:00
---

Recently we launched a [new website](https://cbso.co.uk) for the CBSO built on [Craft CMS](http://buildwithcraft.com), it was a pretty exciting project for us all and was a lot of fun to work on. It was also the first time we got to use our new Box Office system that I’d spent the better part of 3 months building as it needed to display events and integrate with the Spektrix ticketing system to let users purchase tickets.

Whilst that went pretty much to plan one thing that didn’t go particularly well to begin with was the site speed. In this post I’m going to break down the highlights of what we did to take one of the the site’s key pages (the [upcoming concert season](https://cbso.co.uk/whats-on/season/2015-16)) from a whopping __7.92s__ initial load time to a much healthier __1.36__. To begin with I took a look at what was causing the huge load times and to no great surprise it was the server response time or, in geek language, the time to first byte (TTFB). That was a big chunk of the time - __6.32s__. So, getting that down was a clear priority.


## First up; native caching in Craft.
This was a no brainer - as soon as we got close to going live I started working on getting Craft to cache the pages natively. We do this on every site we build now but this time there were a few awkward caveats to watch out for.

### AJAX
A large chunk of the site was loading over AJAX, but using the same urls as the non-AJAX pages. This is nice because it allows us to do things like paginate quicker but still lets people jump in half way through the paginated list. The issue I came up against was how to cache the main page and the AJAXed page separately, but still use the same code. In the end it was quite simple to solve:

I put the twig logic and output in one template:

{% highlight jinja %}
{% raw %}
{# file: events/_logic-and-output #}
<ul>
  {% for event in events %}
  <li>{{ event.title }}
  {% endfor %}
</ul>
{% endraw %}
{% endhighlight %}

Then I put the normal layout extending twig in another:

{% highlight jinja %}
{% raw %}
{# file: events/_output-via-layout #}

{% extends '_layouts/main' %}

{% block main %}
  {% include 'events/_logic-and-output' %}
{% endblock %}


{# file: events/_layouts/main #}

{% cache for 3 hours %}
<html>
  <head>
    <title>Welcome</title>
  </head>
  <body>
    {% block main %}
      <p>Hello world</p>
    {% endblock %}
  </body>
</html>
{% endcache %}

{% endraw %}
{% endhighlight %}

And finally created a little controller template that either just chooses which one to use based on whether the request is an AJAX one or not:

{% highlight jinja %}
{% raw %}
{# file: events/_controller #}

{% if craft.request.isAjax and not craft.request.isLivePreview %}

  {% header "Content-Type: application/json" %}

  {% cache for 3 hours %}
  {% spaceless %}

    {% set html %}{% include 'events/_logic-and-output' %}{% endset %}

    {
      "html" : {{ html | json_encode() | raw }}
    }

  {% endspaceless %}
  {% endcache %}

{% else %}

  {% include 'events/_output-via-layouts' %}

{% endif %}
{% endraw %}
{% endhighlight %}

Suffice to say I was scratching my head for a good few hours and at one point fired an email off to P&T complaining that the cache tag didn’t seem to be improving things, before realising that I had a load of Twig _outside_ the damn thing. Thanks for not screaming at me [Brad](https://twitter.com/angrybrad).

### How long should we cache?
Normally we cache everything for a whole day, and this works fine in most scenarios but for this site we have quite time sensitive information so I ended up caching some sections as low as 3 hours and leaving the rest set to a day.

After implementing native caching we had a new TTFB of __347ms__. A massive reduction from __6.32s__!


## Next up; CDN & browser caching
The next thing I did was to stick all our images and static assets on a CDN. Thankfully Amazon CloudFront makes this pretty easy to do for the static assets we host ourselves - I just followed their guide and it worked really quickly. For our client images we were storing them on Amazon S3 - which is a native feature of Craft. All I had to do there was tell Craft to output the new CDN url instead of the S3 one, which was also trivial.

Finally I used a lot of the .htaccess rules from [this handy template](https://github.com/BarrelStrength/Craft-Master/blob/master/public/.htaccess) by the guys at Barrel Strength to get the browser to cache things properly so repeat views of our test page come in at around 900ms load time.

This all helped with repeat views an awful lot, but obviously didn’t improve our TTFB at all.


## Finally; Varnish

Thankfully André Elvan had already done a bunch of the work for me, so I just used [his config](https://gist.github.com/aelvan/eba03969f91c1bd51c40) as a starting point and then went about [adapting it](https://gist.github.com/joshangell/540eca3cb16590537f54). To begin with I didn’t set a default amount of time to cache things for (time to live or TTL) in Varnish instead relying on setting it as a header in the template. This did work but of course the browser also used that cache time so would require a force reload to clear, which is not ideal in the slightest. To solve this I set my headers from Craft to not cache anything and instead set a default in Varnish, initially to 3 hours to match my lowest Craft cache time.

Craft has a handy `{% header %}` tag which you can use to set the headers of a given template, so I just did this:

{% highlight jinja %}
{% raw %}

{% header "Cache-Control: no-cache" %}
{% header "Pragma: no-cache" %}

{% endraw %}
{% endhighlight %}

### Ignore all the things
Out of the box Andrés setup ignored the Craft admin, any POST requests and removed all cookies by default. This was great as it meant that the whole frontend got cached by Varnish but the backend didn’t and all our forms worked - so far so good. I discovered that removing cookies is important because Varnish won’t work properly if you try and use cookies server side so this threw up one small issue as we we’re using cookies at one point in the site. All I had to do though was set that cookie on the client side using JavaScript and I could move on.

Another thing I had to tell Varnish to ignore was all AJAX requests - if Varnish served a cached AJAX result in our situation it would lead to someone loading a page that was just a mush of json. I found this one out the hard way on the live site ... thankfully the fix was quick and easy, see [line 45 of my config](https://gist.github.com/joshangell/540eca3cb16590537f54#file-default-vcl-L45) for the syntax.

### SSL termination
Sadly Varnish doesn’t work with SSL out of the box and our entire site is served over SSL so this was an important one to solve. After a bit of googling I noticed that nginx offers SSL termination - which means it will receive an SSL request from a client and pass that on as a normal http request to wherever you want it to. Having never used nginx at all there was a bit of a general learning curve for me at first but after I’d got it running in a basic manner I followed [this handy guide](https://www.digitalocean.com/community/tutorials/how-to-configure-varnish-cache-4-0-with-ssl-termination-on-ubuntu-14-04) from DigitalOcean on how to configure nginx correctly so it would pass my https requests to Varnish over http.

Now I have the following stack:

    Browser ->- (SSL) ->- nginx ->- (http) ->- Varnish ->- (http) ->- Apache

At present I have all of this running on one 2GB vps hosted with [Linode](https://linode.com) which seems a little mental to begin with but as it currently works fine I’m not too worried. Further down the line if the traffic really hots up I will likely split the nginx/Varnish setup off onto it’s own server but for now I’m leaving it as is.

Now that we are successfully running Varnish in front of the site I can re-test all my times to see if it was all worth it, and it certainly was. For our test page I now get a TTFB of __25ms__ whereas without Varnish it was __290ms__. The final load is __1.64s__ initially and __638ms__ for repeat views.

### Bust that cache
Having proved that it was worth doing all this I set about solving the final part of the puzzle - how to purge the cache. At this point if a content editor updated an entry and saved it Varnish was just going to keep serving the stale content until either the expiry time was reached or that url was purged. I wanted someone to be able to press save and have the Varnish cache cleared.

Craft has a nice feature that kicks in when you use the native tag caching - after saving a piece of content Craft goes over all the caches it has that are using that content in some way and gets rid of them. As I was already using this I thought I’d just hook into this behaviour and try and purge all the urls in Varnish that were getting cleared out in Craft.

So I wrote a small plugin that retrieved all the cached uri paths for a given content element just before it got saved, then once it had finished saving and Craft had cleared all of it’s own caches for those uris went back over them purging them in Varnish. When implemented as a [Task](http://buildwithcraft.com/classreference/services/TasksService), this worked well. As Craft comes with Guzzle built in I could also take advantage of the batching feature and make my purge requests in batches of 20, which really speeds up the process when there are a lot of uris to go over.

I did come across a couple of caveats - most notably that Guzzle will generate an exception if the page you’ve just tried to request threw an error (in this case some 404s). The solution was just to read the Guzzle docs more thoroughly and collect those exceptions, log them and discard.

### Keeping things quick
One thing I really wanted to avoid was a user ever getting one of those 6 second loads - albeit it wouldn’t happen very often but now I was purging things after saving I was really noticing the difference between a fully cached page and the un-cached one. So, I then went back to my plugin and added to the purging feature a second Task that went back over all those urls that just got purged and made a standard GET request on them, making Varnish cache them afresh.

Now things were really singing - press save, after the Tasks have all run go view the page and its as lightening fast as before. But there was one final niggle - people still had to either visit a page or the editors had to update it before the Varnish TTL was reached if Varnish was to have a cache of it, which meant some pages were still going to get comparatively long waits on them. This irked me as now that the heavily browsed areas of the site were really fast it was noticeable when you went somewhere more obscure that hadn’t been cached.

So I wrote some more code in my plugin to allow me to dump all the template caches, and then crawl the sitemap to get all of the pages we have so I can send them to my page warming Task. This was really exciting the first time I ran it and only takes about 3 or 4 minutes to do over 500 pages. Setting this up with a cron job so that the whole Varnish cache was dumped beforehand was as easy as sticking this in my crontab:

    30 1 * * * (/etc/init.d/varnish restart > /dev/null 2>&1; sleep 45; /usr/bin/curl --silent -H "X-Requested-With:XMLHttpRequest" https://cbso.co.uk/actions/myPlugin/myCrawlingAndWarmingAction)

I had to include the 45s sleep to make sure Varnish had finished restarting otherwise nginx couldn’t get through the stack to Apache and just errored out.

Finally now that we are in complete control of our purging and warming I went back over the full stack and adjusted the default page TTL in Varnish and the equivalent in Craft to 24 hours to keep it all in sync. Over the coming weeks I’ll be monitoring this and evaluating whether we need to tweak this in certain places so its less, particularly when it comes to whether a particular ticket is available to book or not but for now, daily is fine.

## Wrapping up warm
So there we have it - one much speedier website that gets completely refreshed each night but is still able to be updated. Now to solve the last of my current problems - gzipping all the images that are stored on S3 so browsers can download them in a fraction of the time. I’ll write that one up when I’ve worked it out ... feel free to chip in if you have a solution that doesn’t involve manually doing it.

In my final load test I could get 2k concurrent users on the test page over period of 5 minutes without the server falling over. Not bad for a single 2GB instance! Before all of this, even with Craft caching I could only get 200.

To get some of these numbers I used [Flood](https://flood.io) for load testing and a combination of [Webpagetest](http://www.webpagetest.org/) and Chrome dev tools for page load times.

### CacheMonstrr
Yes - that’s what I called the plugin I’ve written to handle the purging and warming side of things. You will be able to download it just as soon as I’ve done the following:

- Allow admins to turn off Varnish purging so that just the warming function is left active
- Allow admins to set the paths of their sitemaps in the cp
