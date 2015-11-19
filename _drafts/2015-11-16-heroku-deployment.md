---
layout: post
title:  "A powerful way to deploy to Heroku"
date:   2015-11-18
comments: true
excerpt: …If we already are here, in this chat, let's also deploy here! And let all stages and the deployment result be obvious. This story tells how I implemented such a scheme.
---

### Prologue

> 11:34:13. **CTO:** Do you deploy?
>
> 11:34:37. **Dev 1:** I do.
>
> 11:36:15. **CTO:** Still deploying?
>
> 11:36:21. **Dev 1:** Yup.
>
> 11:37:13. **CTO:** Deployed?
>
> 11:37:16. **Dev 1:** Wait, it's restarting.
>
> 11:37:22. **CTO:** …Restarted?
>
> 11:37:25. **Dev 1:** Yeah, tell Max he can have a look.
>
> 11:40:01. **CTO:** Max says that nothing has changed. Are you sure it's deployed?
>
> 11:40:48. **Dev 1:** I AM SURE! Works for me. But wait, what the hell…
>
> 11:41:03. **Dev 2:** Oh, guys, are you deploying? I've just deployed my branch.
>
> 11:41:09. **Dev 1:** ...!
>
> 11:41:14. **CTO:** ...!

I am not making this up. Back in 2011 we had such a dialogue in Campfire.

What to say? A process involving many people should be apparent for everyone.
That's why airports provide flight information on big screens, not restricting it
to a flight dispatcher's terminal!

If we already are here, in this chat, let's also deploy here! And let all stages and
the result of deployment be obvious. This story tells how I implemented such a scheme.

Here's how it works in [Shuttlerock](https://www.shuttlerock.com)'s Slack channel:

![Deployment example](/assets/heroku-deploy/slack_rodney_deploy.png){:width="771"}

{% if false %}
Here's more:

![Deployment example 2](/assets/heroku-deploy/slack_rodney_deploy_2.png){:width="768"}
{% endif %}

This solution might be useful for you too!

### Heroku

We tend to use Heroku in our projects. Just as any real thing, Heroku has its pros and contras. It's
more expensive than a virtual server at, say, [DigitalOcean](https://www.digitalocean.com/).
But an ability to sleep at night, while Heroku DevOps engineers
are awake, comes with a price.

Heroku makes the deployment process simple: just a `git push`. After some wait
the app restarts and works; at least in theory. There are some practical nuances, though.

#### Nuance 1. Assets

That's right: CSS, JS & friends. If you don't do anything special about them, your
Heroku app will work. But assets will be served from Ruby:
your expensive web dynos will pose as Apache. First, it's slow (real
Apache and nginx are much, much faster). Second, while Rack code is pushing the compiled
version of `application.js` into a network buffer, incoming requests sit in line and wait.
Such a scheme is apparently only suitable for personal (as in "I'm the only user")
sites.

Therefore most real sites use a [CDN](https://en.wikipedia.org/wiki/Content_delivery_network)
for serving assets. For example, assets can be uploaded
to Amazon S3 which interoperates with Amazon CloudFront, the Amazon's CDN service.
Things are similar with Rackspace
and other cloud platforms. Nothing is better for static files than a CDN – take
this as an axiom.

It must be noted that Rails app needs a manifest file to be able to generate
proper asset URLs. This file is generated during `rake assets:precompile`.
In Rails 4 it's in `public/assets` directory:

{% highlight javascript %}
// public/assets/.sprockets-manifest-9b2e86e85245c42f19250388ba3e1a45.json
{
  "files": {
    "application-3dda9c3d8b35165ec9de63f4b471dc4ce898f16443b1c132aa5049128d8e9a1c.js": {
      "logical_path": "application.js",
      "mtime": "2015-10-07T11:22:21+06:00",
      "size": 341219,
      "digest": "3dda9c3d8b35165ec9de63f4b471dc4ce898f16443b1c132aa5049128d8e9a1c",
      "integrity": "sha256-PdqcPYs1Fl7J3mP0tHHcTOiY8WRDscEyqlBJEo2Omhw="
    },
    // ...
  },
  "assets": {
    "application.js": "application-3dda9c3d8b35165ec9de63f4b471dc4ce898f16443b1c132aa5049128d8e9a1c.js",
    // ...
  }
}
{% endhighlight %}

It's the manifest that allows app to generate that long hexadecimal suffix
when you specify `<%= javascript_include_tag :application %>` in your layout.
If this `application-3dda9c3...yaddayadda.js` file has been uploaded to the CDN,
the generated link would work.

But how does it play with Heroku? By default Heroku runs
`rake assets:precompile` during deploy process. This command populates `public/assets`
directory with compiled asset files, including the manifest. But how could those files make
their way to the CDN? How can we be sure that the manifest is 100% actual?

There's an [asset_sync](https://github.com/AssetSync/asset_sync) gem for that.
It adds a rake task which runs after `rake assets:precompile` and uploads
everything to S3. As such, the synchronization works directly from a Heroku build server
during deploy.

Unfortunately this solution is not perfect. It slows down the deploy process, especially
in large projects. In general, asset precompiler is smart. It caches its results and
wouldn't regenerate things that didn't change. Look at this local experiment:

{% highlight bash %}
rm -rf public/assets
time bin/rake assets:precompile
#> Writing ..... (many lines skipped here)
#> bin/rake assets:precompile  0,19s user 0,10s system 5% cpu 5,191 total
time bin/rake assets:precompile
#> bin/rake assets:precompile  0,16s user 0,07s system 28% cpu 0,807 total
{% endhighlight %}

The second run was a no-op (and thus fast!). But this trick doesn't work with Heroku,
because build server always starts from scratch. Full precompilation and S3 sync
will take place **every time**, even if there were no asset changes. And it hurts!

#### Nuance 2. Migrations

If you use an SQL database, you need to run migrations from time to time. Heroku
doesn't automate this, you have to execute `heroku run rake db:migrate` by youself.
**However**, you can run this command only **after** deployment, when appropriate
files in `db/migrate` directory are in place. There's a possibility that
just deployed Ruby code refers to a non-existent database attribute (the appropriate migration hasn't
been run yet), and an unlucky user gets "500 Internal Error".

To mitigate this problem Heroku suggests to go into maintenance mode before deploying.
The full deployment script will look like this:

{% highlight bash %}
heroku maintenance:on
git push heroku master
heroku run rake db:migrate
heroku maintenance:off
heroku restart
{% endhighlight %}

`heroku restart` is called for the database scheme to be reloaded. In production mode it's loaded
only once, during boot; if there's no `age` attribute in
`users` table, `User#age` and `User#age=` methods won't be created. Even
after the migration runs any code calling these methods will still generate an exception.

Unfortunately, even this protective script doesn't make you 100% safe. There's
still a chance that some unlucky user visits your site between `heroku maintenance:off`
and `heroku restart`. He's still going to get his 500 error. Not to be solved!

Now let's imagine that it takes several minutes for assets to be compiled and synced.
Not surprising for a large app. And we're in maintenance mode during this time!..

### A path to desired system

So far we've described the scope of the problem:

1. Obviousness of deployment process and its results.
2. Assets compilation and uploading to the CDN.
3. Migrations.

Having spent some time thinking, I came to the following solution:

* The deployment process runs inside a [Jenkins](https://jenkins-ci.org/) instance.
* Assets are precompiled and uploaded to S3 **before** Heroku deployment. If there
were no changes since last deploy (as told by Git history), nothing is done.
If something has changed, `rake assets:precompile` can
use local cache to speed up the precompilation process.
* Deployment is triggered by [Hubot](https://hubot.github.com/), a bot which sits
in the chat and interacts with Jenkins. It also reports deploy status.
* The migration run is optional. Two deploy modes are introduced: **regular**
(following the full script mentioned above) and **quick**, when migrations are not
run at all.

The whole system consists of the following parts:

1. A hubot instance.
2. A custom Hubot script which implements `deploy` command and interacts with Jenkins.
3. A Jenkins instance with properly configured jobs.
4. A `bin/deploy.sh` script in the app source tree. This script is run by Jenkins during deployment.
5. A Ruby module inside the app which implements S3 upload.

Let's discuss some of these parts.

#### Hubot

I fell in love with Hubot when I first saw it. I use it all the time now.
My favorite commands are [`image`](https://github.com/hubot-scripts/hubot-google-images) and [`excuse`](https://github.com/github/hubot-scripts/blob/master/src/scripts/excuse.coffee).

Hubot provides a convenient API to your [Slack](https://slack.com/)
([HipChat](https://www.hipchat.com/), [Campfire](https://campfirenow.com/)) chat.
Custom command handlers are relatively easy to implement: it's just Javascript, after all.

Hubot is easily deployed to Heroku.

#### Jenkins

Jenkins is a Java monster. Not my favorite type of apps, but it's very stable and
there's a wide selection of plugins. Writing your own plugin is not as easy as with Hubot,
but there's no need. So, what can Jenkins give us?

* _Git support_. Jenkins can clone and update the repo, and checkout any branch you want.
* _Build history with logs_. If something fails, the log is available to all engineers cool
enough to log in to Jenkins.
* _Job queue_. If two deploy commands were given, the second one would wait until first one finishes.
If the second one is a mistake, you can quickly log in to Jenkins and cancel it.
* _Parallel builds_. If you have several apps or several versions of a single app (staging, production),
deploys can proceed in parallel.

In essence, Jenkins is a fancy shell to run bash scripts working on Git checkouts.

### Configuration

Now let's set up and configure every part. We'll assume that our Heroku app
is named _coolapp_.

#### Jenkins

Jenkins is easy to set up. In Shuttlerock I used [ansible](http://www.ansible.com/)
with [Stouts.jenkins](https://github.com/Stouts/Stouts.jenkins) role, but you can
just install the package. We'll need the following Jenkins plugins:

* [GIT plugin](http://wiki.jenkins-ci.org/display/JENKINS/Git+Plugin).
* [Build Authorization Token Root Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Token+Root+Plugin).
Allows deploy jobs to be triggered by an incoming HTTP request.
* [Notification plugin](https://wiki.jenkins-ci.org/display/JENKINS/Notification+Plugin).
Will notify Hubot about build process.

I also install [Green Balls](http://wiki.jenkins-ci.org/display/JENKINS/Green+Balls) for appearance,
[ChuckNorris Plugin](https://wiki.jenkins-ci.org/display/JENKINS/ChuckNorris+Plugin) for fun,
[Matrix Authorization Strategy Plugin](http://wiki.jenkins-ci.org/display/JENKINS/Matrix+Authorization+Strategy+Plugin)
for convenient access control, [Credentials Plugin](http://wiki.jenkins-ci.org/display/JENKINS/Credentials+Plugin)
for managing additional private keys
and [SCM Sync Configuration Plugin](http://wiki.jenkins-ci.org/display/JENKINS/SCM+Sync+configuration+plugin)
to mirror Jenkins config changes into a Git repo.

The UNIX user under which Jenkins daemon runs (usually just `jenkins`)
should have all needed versions of Ruby installed. Another ansible role,
[zzet.rbenv](https://github.com/zzet/ansible-rbenv-role), proved itself very useful
in this respect.

Now let's move on to creating a deploy job. Let's give it a meaningful name:

![Giving a name to the new job](/assets/heroku-deploy/jenkins_name.png){:width="343"}

Every app version (staging, production, …) will need a separate job. Then configure
notification:

![Notification](/assets/heroku-deploy/jenkins_notify.png){:width="482"}

The URL field must contain the full Hubot URL with `/hubot/jenkins_status` path,
e.g. `https://my-hubot-instance.herokuapp.com/hubot/jenkins_status`.

Further down the page, check the "Parameterized Build" checkbox. The parameters
we are going to create will be available to a deploy script as environment
variables. We'll also be able to pass their values from outside upon triggering
a build with HTTP request. The following parameters should be added:

|-----------+-----------+---------------------+--------------|
| Name      | Type      | Default Value       | Description     |
|-----------+-----------+---------------------+--------------|
| APP       | String    | `coolapp-staging`   | Heroku application name |
| BRANCH    | String    | `master`            | Git branch Name |
| DEPLOYER  | String    | _(empty)_           | Deployer name |
| QUICK     | Boolean   | Off                 | Quick mode doesn't run migrations |
| EXTRA     | String    | _(empty)_           | Extra parameters |
|-----------+-----------+---------------------+--------------|
{: .table}

Select Git in "Source Code Management", provide a URL of your repo and
put `origin/$BRANCH` into the branch field:

![SCM Configuration](/assets/heroku-deploy/jenkins_scm.png){:width="544"}

This will allow us to deploy any branch present in the repo.

Now specify a remote token in "Build Triggers" and uncheck other checkboxes:

![Build Triggers](/assets/heroku-deploy/jenkins_build_trigger.png){:width="422"}

What's left is adding a single build step ("Execute Shell"):

{% highlight bash %}
# Use dynamic Ruby version. Don't forget to specify it in your Gemfile!
# E.g.
#   ruby '2.2.3'
ruby_version=`grep '^ruby' Gemfile|cut -d "'" -f 2`

# The home directory of jenkins user is /var/lib/jenkins. Change if yours is different.
export PATH=/var/lib/jenkins/.rbenv/versions/$ruby_version/bin:/var/lib/jenkins/.rbenv/shims:$PATH
export HOME=/var/lib/jenkins

# Make EXTRA parameter available inside bin/deploy.sh
export EXTRA

# S3 credentials used to upload assets:
export AWS_ACCESS_KEY_ID=<key>
export AWS_SECRET_ACCESS_KEY=<secret-key>
export AWS_BUCKET=coolapp-prod-assets

# Put the environment name here:
export RAILS_ENV=staging

# In some Rails versions `rake assets:precompile` needs a database connection (!).
# It was easier to provide one than struggle; I've created an empty database
# accessible by jenkins user with jenkins password.
export DATABASE_URL=postgresql://jenkins:jenkins@127.0.0.1/jenkins_empty

# I've also stumbled upon Devise requiring the secret key. Hey, we're just
# doing rake assets:precompile!.. :(
export DEVISE_SECRET_KEY=fb02df94e6fb4

# These commands are helpful if something doesn't work. Uncomment if you debug.

#env
#gem env
#bundle env

if [[ -f bin/deploy.sh ]]; then
    exec bin/deploy.sh
else
    false
fi
{% endhighlight %}

So, we're setting everything up and delegating actual deployment to
the `bin/deploy.sh` script.

Note that if you're creating a second, third, etc. job, don't start from
scratch, use copy mode instead:

![Copy the existing job](/assets/heroku-deploy/jenkins_copy.png){:width="725"}

Now let's move on to Hubot.

#### Configuring Hubot

Install Hubot following the [instruction](https://hubot.github.com/docs/). By the way,
pick a catchy and fun name for your bot instead of "hubot"! E.g. in Shuttlerock we have
_rodney_.

Add [deploy.coffee](https://gist.github.com/be9/87727f2f41c8709036e2#file-deploy-coffee)
to the `scripts` directory. This version works well for Slack; if you use other adapters,
the script might need minimal changes.

Then you'll have to edit a single line in the `deploy.coffee` file:

{% highlight coffee %}
APPS = ['production', 'staging']
{% endhighlight %}

Specify all apps/app versions which you are going to deploy here. E.g. this
line could look like this:

{% highlight coffee %}
APPS = ['production', 'staging', 'monitoring production', 'monitoring staging']
{% endhighlight %}

This assumes two versions of the main app and two versions of the monitoring app.

Other configuration is done by setting environment variables for the Hubot instance.
Let's look at what's needed.

1. The `HUBOT_JENKINS_URL` variable must contain the root Jenkins URL (e.g.,
`https://jenkins.example.com`).

2. The `HUBOT_JENKINS_BUILD_TOKEN` contains the token which you put into
the job configuration (`0W5CT73cFV4ia89N9Sa87S644v3twA9P` in our example).
It's assumed that this token is the same for all apps and app versions.

3. Now set 4 variables for each app from `APPS`:
* `HUBOT_PRODUCTION_APP`. The name of Heroku app. `coolapp-staging` in our case.
* `HUBOT_PRODUCTION_DEFAULT_BRANCH`. The default branch to be deployed (when name is not specified). E.g. `master`.
* `HUBOT_PRODUCTION_JOB`. The Jenkins job name. `coolapp-staging-deploy` for us.
* `HUBOT_PRODUCTION_ACL`. This variable manages access to deployment. It should either
contain the list of E-mails of admitted people or be set to `everyone`.
Note that you can see the full E-mail list with `hubot show users` command.

Variable names are obtained from the app name. E.g. _monitoring staging_
has `HUBOT_MONITORING_STAGING_APP`, `HUBOT_MONITORING_STAGING_DEFAULT_BRANCH` etc.

At this point if everything has been configured the right way, a bot command (`hubot deploy to staging`)
will trigger a Jenkins build with proper parameters. Now let's look at
the deployment core, `bin/deploy.sh`.

#### bin/deploy.sh

The script is provided in the same [gist](https://gist.github.com/be9/87727f2f41c8709036e2#file-deploy-sh);
here we are going to examine how it works, function by function.

**set_extra_flags**

{% highlight bash %}
set_extra_flags() {
  if [[ "$EXTRA" =~ 'reupload assets' ]]; then
    export CLOUD_ASSETS_REUPLOAD=1
  fi

  if [[ "$EXTRA" =~ 'recompile assets' ]]; then
    export CLOUD_ASSETS_RECOMPILE=1
  fi

  if [[ "$EXTRA" =~ 'cleanup assets' ]]; then
    export CLOUD_ASSETS_REMOTE_DELETE=1
  fi

  if [[ "$EXTRA" =~ 'skip heroku' ]]; then
    export SKIP_HEROKU=1
  fi

  if [[ "$EXTRA" =~ 'clear cache' ]]; then
    export CLEAR_CACHE=1
  fi
}
{% endhighlight %}

This shows how a bot command's "tail" is used: you can pass additional
parameters in there. `hubot deploy to staging` is the basic story, but
`hubot deploy to staging and recompile assets and clear cache` does more:
`CLOUD_ASSETS_RECOMPILE` and `CLEAR_CACHE` variables
influence the process (see further).

**compile_assets**

{% highlight bash %}
compile_assets() {
  current_sha=`git log -n 1 --pretty=format:%H app/assets vendor/assets`

  if [[ -f public/assets/CURRENT_SHA && `cat public/assets/CURRENT_SHA` == $current_sha && "$CLOUD_ASSETS_REUPLOAD" == '' && "$CLOUD_ASSETS_RECOMPILE" == '' ]]; then
    echo "Assets did not change (SHA $current_sha)"
  else
    echo "Recompiling assets"

    bundle install --quiet --without=test

    rm -rf public/assets

    time bundle exec rake assets:precompile cloud_assets:sync

    echo $current_sha > public/assets/CURRENT_SHA
  fi
}
{% endhighlight %}

The `public/assets/CURRENT_SHA` file contains SHA1 of the latest Git commit
that changed `app/assets` or `vendor/assets` (if you have other asset folders,
add them as additional `git log` arguments).

If nothing has changed since then, no compilation would occur. In other case
precompilation and S3 upload are triggered (`rm -rf public/assets` is done just in case, it's
not required).

For S3 upload to work, add [lib/tasks/cloud_assets.rake](https://gist.github.com/be9/87727f2f41c8709036e2#file-cloud_assets-rake)
and [lib/fog_cloud_assets.rb](https://gist.github.com/be9/87727f2f41c8709036e2#file-fog_cloud_assets-rb).
to your source tree. For latter module to work you'll also need the `fog_aws` gem; don't
forget to add it to your `Gemfile`:

{% highlight ruby %}
gem 'fog-aws', require: 'fog/aws'
{% endhighlight %}

The FogCloudAssets module does an incremental S3 upload.
If `recompile assets` mode, as we already saw, triggers full recompilation,
`reupload assets` does a full S3 upload: all files are reuploaded even if they
are already on S3. `cleanup assets` removes all assets on S3 which are missing
from the current manifest.

**save_deploy_information** and **commit**

We manage assets before deployment, but how can we make sure that the correct
manifest is present in the deployed code? I solved this problem in a following way:
a temporary branch is created on every deploy; the manifest file is added and committed
to Git; then `git push --force` is run. Heroku sees the committed manifest
and skips the `rake assets:precompile` step.

It's not very elegant, but brings new possibilities! The `save_deploy_information`
function, for example, generates a `lib/deploy_info.rb` file which has the following format:

{% highlight ruby %}
module DeployInfo
  BRANCH='feature/5852-core-can-upload-to-boards-via-the-api-when-submissions-false'
  GIT_COMMIT='833a167e8bfd747816eb7337a45394f288ce813f'
  BUILD_NUMBER='1217'
  BUILD_ID='1217'
  DEPLOYER='dave'

  def message
    return @message if defined?(@message)
    text = File.read(__FILE__)
    text =~ /[_]_END__(.*)$/m
    @message = ($1 || '').strip
  end

  module_function :message
end
__END__
commit 833a167e8bfd747816eb7337a45394f288ce813f
Author: John Doe <johndoe@example.org>
Date:   Thu Nov 12 18:02:04 2015 +0300

    #5852: Updated specs for Api::V1::BoardItemsController
{% endhighlight %}

This makes what's currently deployed apparent. We use
[ActiveAdmin](https://github.com/active_admin/active_admin):

{% highlight ruby %}
# app/admin/dashboards.rb
ActiveAdmin.register_page "Dashboard" do
  content do
    # ...

    columns do
      # ...

      column do
        panel "Deploy Information" do
          require 'deploy_info'

          github = "https://github.com/CoolCompany/coolapp/"

          attributes_table_for DeployInfo do
            row('Branch')       { link_to DeployInfo::BRANCH, "#{github}tree/#{DeployInfo::BRANCH}" }
            row('Commit')       { link_to DeployInfo::GIT_COMMIT, "#{github}commit/#{DeployInfo::GIT_COMMIT}" }
            row('Build Number') { DeployInfo::BUILD_NUMBER }
            row('Build ID')     { DeployInfo::BUILD_ID }
            row('Deployer')     { DeployInfo::DEPLOYER }
            row(:message)       { pre DeployInfo.message }
          end
        end # panel
      end # column
    end # columns
  end # content
{% endhighlight %}

The panel looks quite nice and informative:

![Deploy information in ActiveAdmin](/assets/heroku-deploy/active_admin_deploy_info.png){:width="1035"}

For it to work locally, add the following `lib/deploy_info.rb` to your source tree:

{% highlight ruby %}
# NOTE: This file is overwritten during deployment!
module DeployInfo
  BRANCH=`git rev-parse --abbrev-ref HEAD`.strip
  GIT_COMMIT=`git rev-parse HEAD`.strip
  BUILD_NUMBER='dev'
  BUILD_ID='dev'
  DEPLOYER=`git config user.name`.strip

  def message
    `git log -1 --pretty=medium`.strip
  end

  module_function :message
end
{% endhighlight %}

After a tour through details, let's consider the main deployment logic.

**bin/deploy.sh: main logic**

{% highlight bash %}
set_extra_flags
compile_assets
save_deploy_information
commit

if [ "$QUICK" = "true" ]; then

    if [[ "$SKIP_HEROKU" == '' ]]; then
      git_push
    else
      echo "Skipping heroku push as requested"
    fi

    #if [[ "$CLEAR_CACHE" == '1' ]]; then
    #  echo "Clearing cache per request"
    #  heroku run rake cache:clear --app $APP
    #fi

    echo "QUICK mode, not running migrations"
else
    if [[ "$SKIP_HEROKU" == '' ]]; then
      heroku maintenance:on --app $APP

      git_push

      heroku run rake db:migrate --app $APP #cache:clear db:migrate --app $APP

      heroku maintenance:off --app $APP

      heroku restart --app $APP
    else
      echo "Skipping heroku push as requested"
    fi
fi
{% endhighlight %}

Depending on `QUICK` param, we follow either short or long path. Other things
are also shown here:

* `skip heroku` mode allows to precompile and upload assets, but skip the Heroku push. Rarely used, but sometimes helpful.
* We have `rake cache:clear` [call `Rails.cache.clear`](https://gist.github.com/be9/87727f2f41c8709036e2#file-cache-rake)
in Shuttlerock. The commented out part shows how this can be used.
Full deploy proactively runs `cache:clear` along with `db:migrate`,
while quick mode allows you to tell `... and clear cache` to the bot.

Following this logic you can easily add your own deploy flags and modes.
Neither Hubot, nor Jenkins is to be touched; you'll only have to handle
stuff in `bin/deploy.sh`.

### The deploy command instructions

Whew, you've set everything up. What can you do now?

`hubot deploy to production` deploys the production app (default branch).

`hubot quick deploy feature/something-really-cool to staging` deploys the specified branch (`feature/…`) to staging
in quick mode; no migrations are run.

`hubot deploy to production and recompile assets` deploys the default branch to production,
forcefully recompiling assets even if there was no change.

`hubot disable deploys to staging` temporary disables staging deploys (useful
if you do maintenance or are debugging something and don't want
anybody to interfere). A reason can be specified:
`hubot disable deploys to staging because it hurts`.

`hubot enable deploys to staging` reenables deploys.

Note that the enable/disable feature works best if you add a Redis "brain" to your Hubot:
see the  [hubot-redis-brain](https://github.com/hubot-scripts/hubot-redis-brain) plugin.

### Conclusion

What we have now is a classy modular deployment system. Its advantages for Rails:

* Convenient, manageable Heroku deploys.
* Asset compilation is a separate step, not slowing down the Heroku push.
No asset changes – no recompilation/sync!

But the system is not specifically tied to Rails. It also has general advantages for
all technologies:

* Everyone sees what's happening.
* There's a history of successful and failed deploys.
* Everyone who's allowed can deploy. Including novices and testers who have
hard time with setting up and updating local dependencies (Heroku Toolbelt,
Ruby, bundler, node.js, etc.)
* Any Git branch can be deployed.
* Any technology can be supported by writing a proper script.

In Shuttlerock all active projects are deployed using this system. E.g. we wrote
a script for an [angular.js](https://angularjs.org/) app using [bower](http://bower.io/)
and [gulp](http://gulpjs.com/).

My colleagues tell me that it's the best deployment system ever. I trust their
word 😄. Why don't you try?
