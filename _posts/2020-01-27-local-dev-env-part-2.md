---
layout: post
title: "Local Development Environments, Part 2"
description: ""
category:
---

In [part 1](/2020/01/26/local-dev-env-part-1.html), we discussed using [inpsec](https://inspec.io)
to validate configuration. Now, we'll tackle how to know if one is running the latest version of
our tools.

We developed a tool named `remy` which is just a thin-veneer over various chef development tools.
It is comprised of several gems that codify in-house standards.

This first command implemented was `version` which displays all the version strings.

~~~
$ remy version
Using remy 2.2.0
Using remy-chefspec 1.9.0
Using remy-foodcritic 0.3.0
Using remy-rubocop 0.5.0
Using remy-kitchen 0.16.0
~~~

It was a succinct way to get all the versions back from a user. They could just cut and paste the
output in slack if we were diagnosing an issue.

A lot of times, the issue was that the user was running an older version of some component. As we
wanted to make things more self-service, we needed to programmatically answer the question of whether
the user was running the latest bits.

~~~
$ remy version --check
Using remy 2.2.0, latest 2.2.2
Using remy-chefspec 1.9.0
Using remy-foodcritic 0.3.0
Using remy-rubocop 0.5.0
Using remy-kitchen 0.16.0
~~~

The code that does this is:

~~~ ruby
all.map do |name, version|
  if version.nil?
    say "#{name} not installed", :yellow
    next
  end

  version = Gem::Version.new(version)

  min = if options[:check]
    Gem.latest_version_for(name)
  end

  if !min.nil? && version < min
    say "Using #{name} #{version}, latest #{min}", :yellow
  else
    say "Using #{name} #{version}", :cyan
  end
end
~~~

Since `Gem.latest_version_for` is rather slow, we only do that when `--version` is passed.

Now circling back to inpsec, we wrote a `remy` control:
~~~ ruby
control 'remy' do
  title 'Verify remy gem is installed and up to date.'
  versions = command('chef exec remy version --check')
  describe versions do
    its('exit_status') { should eq 0 }
  end

  versions.stdout.split("\n").each do |version|
    next unless version =~ /remy/
    describe version do
      it { should_not match(/latest/) }
    end
  end
end
~~~

And since `remy` is out-of-date, it will fail.
~~~
$ inspec exec remy.rb

Profile: tests from remy.rb
Version: (not specified)
Target:  local://

  ×  remy: Verify remy gem is installed and up to date. (1 failed)
     ✔  Command chef exec remy version --check exit_status should eq 0
     ×  Using remy 2.2.0, latest 2.2.2 should not match /latest/
     expected "Using remy 2.2.0, latest 2.2.2" not to match /latest/
     Diff:
     @@ -1,2 +1,2 @@
     -/latest/
     +"Using remy 2.2.0, latest 2.2.2"

     ✔  Using remy-chefspec 1.9.0 should not match /latest/
     ✔  Using remy-foodcritic 0.3.0 should not match /latest/
     ✔  Using remy-rubocop 0.5.0 should not match /latest/
     ✔  Using remy-kitchen 0.16.0 should not match /latest/

Profile Summary: 0 successful, 1 failures, 0 skipped
Test Summary: 5 successful, 1 failures, 0 skipped
~~~

In part 3, we'll discuss how to self-update.
