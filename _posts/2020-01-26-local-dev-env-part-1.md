---
layout: post
title: "Local Development Environments, Part 1"
description: ""
category:
---

At DeliveryConf 2020, I facilitated a discussion about local development environments after
[Patrick Gray's](https://www.deliveryconf.com/speakers/patrick-gray/) session
[Localdev And A Tight Feedback Loop](https://www.deliveryconf.com/talks/localdev-and-tight-feedback-loop/).

I've participated in several discussions over the years. Why are local development environments so
hard?

I have spent countless hours diagnosing various people's environments. Then I had the epiphany:
Treat localdev as you would production.

And what do I mean by that? Use the same tools that you are familiar with and use for production
and apply that to local dev.

We use [inspec](https://www.inspec.io) to validate configuration on our production servers. For
local dev, we need various things configured. Why not use inspec to verify required configuration.

For example, we want to pull gems from our artifactory instance instead of https://rubygems.org.

~~~ ruby
control 'gemrc' do
  title 'Verify $HOME/.gemrc'

  path = "#{ENV['HOME']}/.gemrc"

  describe file(path) do
    it { should be_readable }
  end

  describe yaml(path) do
    describe ':sources' do
      subject { YAML.load_file(path)[:sources] }
      it { should_not be_empty }
      it { should_not include(/rubygems.org/) }
      it { should include(/artifactory.nvwls.com/) }
    end
  end
end
~~~

Now instead of a back and forth or slack, "What does your .gemrc file look like?" you can just have
them run easily verify their environment with inspec.

~~~
$ inspec exec localdev.rb

Profile: tests from localdev.rb
Version: (not specified)
Target:  local://

  ✔  gemrc: Verify $HOME/.gemrc
     ✔  File /Users/joe.nuspl/.gemrc should be readable
     ✔  YAML /Users/joe.nuspl/.gemrc :sources should not be empty
     ✔  YAML /Users/joe.nuspl/.gemrc :sources should not include /rubygems.org/
     ✔  YAML /Users/joe.nuspl/.gemrc :sources should include /artifactory.nvwls.com/

Profile Summary: 1 successful, 0 failures, 0 skipped
Test Summary: 4 successful, 0 failures, 0 skipped
~~~

At first, I was hesitant to use .gemrc as the example because `:sources` is a symbol key in .gemrc.
inspec does not handle symbol keys so I had to override the `subject` of the block. In the end I
I thought others might has similar set ups so providing a working example for this corner case
would be helpful.

As a chef shop, we have controls for .gemrc, bundler, Berkshelf, docker, and various other
configurations. The controls grew organically. We started with a few things and whenever we
encountered a new issue, we wrote a new control to verify a configuration. Actually, a lot of the
tests are of the `should_not` variety. For example, certain versions of gems were known to be
buggy so we verified those were not installed.

This work was inspired by the `brew doctor` command from (Homebrew)[https://brew.sh].

Stayed tuned for part two where we tackle the question: Are you running the latest version?
