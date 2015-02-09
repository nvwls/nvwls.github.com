---
layout: post
title: "zap-directory-now-supports-recursive"
description: ""
category:
tags: []
---
{% include JB/setup %}

[https://github.com/nvwls/zap] is a library cookbook for garbage collecting chef
controlled resource sets. For those unfamilar with zap, please check out my
ChefConf 2014 presentation
[http://www.youtube.com/watch?v=4-So4AJlBI4&list=PL11cZfNdwNyMmx0msapJfuGsLV43C7XsA&feature=share&index=53]
and the slide deck
[https://speakerdeck.com/nvwls/building-authoritative-resource-sets]

A month ago, [https://github.com/withnale] raised the issue that `zap_directory`
would fail if it encountered subdirectory within the directory you were
zapping. In v0.8.0, I fixed that bug. It will skip directories it encounters.
At the same, I also added the `recursive` attribute to `zap_directory`. When set
to true, all files under the specified directory will be zapped.

I've given you a very sharp stick, don't hurt yourself with it.

~~~ ruby
zap_directory "#{eval_to_nil}/etc" do
  recursive true
end
~~~

This would zap most of /etc.  For this reason, when I use `zap_directory`
I always use an explict directory name instead of an evaluated one.

### An Example

Let's build a recipe for a hypothectical application called *crossbar*. It has a configuration structure of:

~~~
/etc/crossbar/*.conf
/etc/crossbar/inputs/*.conf
/etc/crossbar/outputs/*.conf
~~~

Since different nodes will have different inputs and outputs, node attributes
will be used to control which conf files are installed.  Optionally there is a
list of users and groups that have access to crossbar.

In the past you might have done something like this:

~~~ ruby
template '/etc/crossbar/groups.conf' do
  if node['crossbar']['groups'].empty?
    action :delete
  else
    action :create
    source 'groups.conf'
    variables 'groups' => node['crossbar']['groups']
  end
end

%w[foo bar baz].each do |src|
  template "/etc/crossbar/inputs/#{n}.conf" do
    if node['crossbar']['inputs'].include?(src)
      action :create
      source "inputs/#{src}.conf"
    else
      action :delete
    end
  end
end
~~~

Every time you add an input or output type, you'd need to update the lists.

Using zap, you don't have to worry about this anymore; just create the resource
if you need it.

The final recipe may look something like:

~~~ ruby
package 'crossbar' do
  version node['crossbar']['version']
end

# installed by crossbar RPM
file '/etc/crossbar/defaults.conf' do
  action :nothing
end

template '/etc/crossbar/groups.conf' do
  source 'groups.conf'
  variables 'groups' => node['crossbar']['groups']
end unless { node['crossbar']['groups'].empty? }

template '/etc/crossbar/users.conf' do
  source 'users.conf'
  variables 'groups' => node['crossbar']['users']
end unless { node['crossbar']['users'].empty? }

node['crossbar']['inputs'].each do |src|
  template "/etc/crossbar/inputs/#{src}.conf"
    source "inputs/#{src}.erb"
  end
end

node['crossbar']['outputs'].each do |dst|
  template "/etc/crossbar/outputs/"{dst}.conf"
    source "outputs/#{dst}.erb"
  end
end

zap_directory '/etc/crossbar' do
  filter '*.conf'
  recursive true
end
~~~

The declarative model is easier to understand and maintain. If previous versions
of the package laid additional conf files, you would need to maintain the list
of files to delete going forward.  You don't know if the node was previously
running v1.0, v1.1, or v2.0 you need to cover all the bases.

Hopefully this example helps illustrate the power of the zap paradigm.

