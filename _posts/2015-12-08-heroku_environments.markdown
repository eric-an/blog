---
layout: post
title:  "Heroku: deploying in multiple environments"
date:   2015-12-08
permalink: /heroku_environments
---

A common question comes up on how to deploy the same app to two different remote repositories on Heroku - for instance, a "staging" remote and a "production" remote.

**Creating two remotes from scratch:**
---

1. `$ heroku create --remote staging`
2. `$ heroku create --remote production`<br>
*This will create two separate heroku apps, each one linked to a git remote named `staging` and one named `production`.*

{% highlight ruby linenos %}
$ heroku create --remote staging
  Creating arcane-river-5903... done, stack is cedar-14
  https://arcane-river-5903.herokuapp.com/ | https://git.heroku.com/arcane-river-5903.git
  Git remote staging added
{% endhighlight %}

{% highlight ruby linenos %}
$ heroku create --remote production
  Creating stormy-earth-1976... done, stack is cedar-14
  https://stormy-earth-1976.herokuapp.com/ | https://git.heroku.com/stormy-earth-1976.git
  Git remote production added
{% endhighlight %}

(To push your app to the arcane-river-5903 app, run `$ git push staging master`)
(To push your app to the stormy-earth-1976 app, run `$ git push production master`)

{% highlight ruby linenos %}
$ git remote -v
  production  https://git.heroku.com/stormy-earth-1976.git (fetch)
  production  https://git.heroku.com/stormy-earth-1976.git (push)
  staging https://git.heroku.com/arcane-river-5903.git (fetch)
  staging https://git.heroku.com/arcane-river-5903.git (push)
{% endhighlight %}

**Copying an existing remote to a new remote:**
---

1. `$ heroku fork --from existingappname --to newappname`
*This will copy all of the config vars, add-ons and Postgres data to the new app.*

{% highlight ruby linenos %}
$ heroku fork --from limitless-anchorage-9300 --to desolate-plateau-9726
  Forking limitless-anchorage-9300... done. Forked to desolate-plateau-9726
  Setting buildpacks... done
  Deploying 3072e33 to desolate-plateau-9726... done
  Adding addon heroku-postgresql:hobby-dev to desolate-plateau-9726... done
  Transferring DATABASE to DATABASE...
  Progress: done                      
  Copying config vars:
    LANG
    RAILS_ENV
    RACK_ENV
    SECRET_KEY_BASE
    RAILS_SERVE_STATIC_FILES
    ... done
  Fork complete. View it at https://desolate-plateau-9726.herokuapp.com/
{% endhighlight %}

2. `$ heroku git:remote -a newappname -r gitremotename`
*This will create a new git remote (production) that links to the new app (desolate-plateau-9726).*

{% highlight ruby linenos %}
$ heroku git:remote -a desolate-plateau-9726 -r production
  set git remote production to https://git.heroku.com/desolate-plateau-9726.git
{% endhighlight %}

* Note: If you want to change that remote to be "staging" instead of "heroku", here is one simple method:
<ol>
  <li><code>$ git remote rm heroku</code></li>
  <li><code>$ heroku git:remote -a limitless-anchorage-9300 -r staging</code></li>
</ol>