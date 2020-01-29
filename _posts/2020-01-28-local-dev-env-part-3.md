---
layout: post
title: "Local Development Environments, Part 3"
description: ""
category:
---

In [part 2](/2020/01/27/local-dev-env-part-2.html), we discussed how to know if you're running the
latest version. Now, we'll tackle how to self update.

So based on the version check, we know we're out of date:
~~~
$ remy version --check
Using remy 2.2.0, latest 2.2.2
Using remy-chefspec 1.9.0
Using remy-foodcritic 0.3.0
Using remy-rubocop 0.5.0
Using remy-kitchen 0.16.0
~~~

We want a simple command that will update all of our component gems.

~~~
$ remy update
         run  gem install remy -v2.2.2 from "."
Fetching: remy-2.2.2.gem (100%)
Successfully installed remy-2.2.2
1 gem installed
~~~

The code that does this is:

~~~ ruby
desc 'update', 'Update remy gems'
def update
  VERSIONS.map do |name, version|
    next if name !~ /^remy/

    version = Gem::Version.new(version)
    latest = Gem.latest_version_for(name)

    if version < latest
      # This gem is out of date
      run "gem install #{name} -v#{latest}"
    end
  end
end
~~~

So you may be thinking, "well the code is pretty simple, why does this warrant a blog post?"

It is simple and that is the point. There are a lot of small things that we can do to improve the
user experience of our tools. Sure, we could just document the update procedure. But what if the
process goes beyond one step or becomes conditional i.e. if you need version X on OSX and version Y
on Linux. This complicates the documentation and when you read it, it sounds a lot like code. Plus
show me a developer that would rather write docs than code...

The update documentation is simply "run `remy update`".

Years ago I read a quote that went something like "if you can document it, you can automate it."
This really had a profound impact on the way I approach problems so thank you, whoever wrote that.

We would not hesitate to automate a task that we need to do across 5/10/50 machines. Let's not
forget about automating that "simple" task across 50 developers.

Remember the tools we build are just a means to an end for our users. Any time they spend updating,
debugging, complaining about the tool is time *NOT* spent on doing their job (and providing value to
their business units).
