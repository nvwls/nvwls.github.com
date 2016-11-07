---
layout: post
title: "Building an Omnibus Ruby"
description: ""
category:
tags: []
---

At the 2016 Chef Community Summit, I attended the "Air-Gapped
Environments" session. Although I do not work in an air-gapped
environment, our environment has a couple of similar restrictions:

* No internet access
* No compilers

Our solution was to build an omnibus ruby, installing all the gems on
a build server that does have has compilers. Therefore when the
package was installed the binary gem, mysql2, already installed.

"Cool idea!" was one response. "How'd you do that?" was another. In
the true spirit of open spaces, my response was to suggest that they
propose an "Omnibus" open space the following day.

The next day during the "Omnibus" open space, I was again describing
our approach. People thought it was a clever solution.

"So how do you keep all the gem dependencies up-to-date?"

"I don't" was my response. "I check in a Gemfile and let bundler
handle all that for me."

There was one mind blown gesture.

When I first coded it, I thought that solution was obvious but perhaps
not.

In `config/Gemfile` I pin the specific version of gems that are
required:

{% highlight ruby %}
source 'https://rubygems.org'

gem 'json', '2.0.2'
gem 'mysql2', '0.3.7'
gem 'net-ldap', '0.3.1'
gem 'pry', '0.10.4'
gem 'zip', '2.0.2'
{% endhighlight %}

The only thing I have in `config/software` is `gems.rb`:

{% highlight ruby %}
name 'gems'
default_version '0.0.0'

skip_transitive_dependency_licensing true

build do
  env = with_standard_compiler_flags(with_embedded_path)

  bundle "install --gemfile=#{Omnibus::Config.project_root}/config/Gemfile", env: env
end
{% endhighlight %}

My `config/projects/ruby.rb` is fairly straight forward:

{% highlight ruby %}
name 'ruby-2.3.1'
maintainer '**REDACATED**'
homepage '**REDACTED**'

install_dir "#{default_root}/#{name}"

require_relative '../../version.rb'
build_version	$VERSION
build_iteration	$RELEASE

override 'ruby', version: '2.3.1', source: { md5: '0d896c2e7fd54f722b399f407e48a4c6' }

dependency 'ruby'
dependency 'bundler'
dependency 'gems'

exclude '**/.git'
exclude '**/bundler/git'
{% endhighlight %}

I am pulling in `omnibus` and `omnibus-software` in my project's
`Gemfile` to get the definitions for `ruby` and `bundler`.

As you can see, with very little code, we can build an omnibus ruby
package containing all of the gems required.
