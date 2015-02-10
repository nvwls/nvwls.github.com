---
layout: post
title: "Testing hardware-specific recipes with chefpsec"
description: ""
category:
tags: ["chefspec"]
---
{% include JB/setup %}

One of the complaints I hear about chefspec is that you cannot test
hardware-specific recipes with chefspec.  Oh, but you can.

Remember, chefsepc is about testing the logic in your recipes.  Did
the correct resources get created?

Given the following recipe:

~~~ ruby
require 'chef/sugar'

case node.deep_fetch('dmi', 'system', 'manufacturer')
when 'Dell Inc.'
  package 'raidcfg'

when 'HP'
  package 'hpacucli'
end
~~~

You can write chefspec test like:

~~~ ruby
require 'chefspec'

describe 'raid::utils' do
  context 'on Dell' do
    let :runner do
      ChefSpec::Runner.new do |node|
        node.automatic['dmi']['system']['manufacturer'] = 'Dell Inc.'
      end.converge(described_recipe)
    end

    it 'converges' do
      expect(runner).to install_package('raidcfg')
      expect(runner).not_to install_package('hpacucli')
    end
  end

  context 'on HP' do
    let :runner do
      ChefSpec::Runner.new do |node|
        node.automatic['dmi']['system']['manufacturer'] = 'HP'
      end.converge(described_recipe)
    end

    it 'converges' do
      expect(runner).not_to install_package('raidcfg')
      expect(runner).to install_package('hpacucli')
    end
  end
end
~~~

This works but it is a rather short sided approach. What happens in
six months when you need to support another hardware vendor?  You
would need to go back and update all the spec tests in all of the
cookbooks.

A better way of doing this using real data. I have a gem CSH (Chef
Spec Helpers) that contains ohai data from all of the hardware
platforms in our data centers.

~~~
$ ls -1 hardware
ESX_Guest.json
Intel_D945GCLF2.json
OpenStack_Guest.json
PowerEdge_M620.json
PowerEdge_M710HD.json
PowerEdge_R310.json
PowerEdge_R320.json
PowerEdge_R410.json
PowerEdge_R420.json
PowerEdge_R510.json
PowerEdge_R720.json
PowerEdge_R720xd.json
ProLiant_DL360e_Gen8.json
ProLiant_DL360p_Gen8.json
ProLiant_DL380p_Gen8.json
VirtualBox_Guest.json
Xen_Guest.json
~~~

The excerpt from CSH:

~~~ ruby
# encoding: UTF-8

module CSH
  HARDWARE = {
    dell: %w[
      PowerEdge_M620
      PowerEdge_M710HD
      PowerEdge_R310
      PowerEdge_R320
      PowerEdge_R410
      PowerEdge_R420
      PowerEdge_R510
      PowerEdge_R720
      PowerEdge_R720xd
    ],
    hp: %w[
      ProLiant_DL360e_Gen8
      ProLiant_DL360p_Gen8
      ProLiant_DL380p_Gen8
    ],
    other: %w[
      Intel_D945GCLF2
    ],
    virtual: %w[
      ESX_Guest
      OpenStack_Guest
      VirtualBox_Guest
      Xen_Guest
    ]
  }

  # iterate over specified platforms
  def self.each_hardware(*kinds, &block)
    array = []
    kinds = HARDWARE.keys if kinds.empty?
    kinds.each do |k|
      fail "Unknown kind #{k.inspect}" unless HARDWARE.include?(k)
      array.concat HARDWARE[k]
    end
    array.each do |desc|
      file = File.join(File.dirname(__FILE__), 'hardware', "#{desc}.json")
      ohai = JSON.parse(IO.read(file))
      yield desc, ohai
    end
  end
end
~~~~

The chefspec tests then become:

~~~ ruby
require 'chefspec'
require 'csh'

describe 'raid::utils' do
  CSH.each_hardware do |machine, json|
    context "on #{machine}" do
      let :runner do
        ChefSpec::Runner.new do |node|
          node.automatic.merge! json['automatic']
        end.converge(described_recipe)
      end

      it 'converges' do
        if machine =~ /^PowerEdge/
          expect(runner).to install_package('raidcfg')
        else
          expect(runner).not_to install_package('raidcfg')
        end

        if machine =~ /^ProLiant/
          expect(runner).to install_package('hpacucli')
        else
          expect(runner).not_to install_package('hpacucli')
        end
      end
    end
  end
end
~~~

This while iterate over all hardware platforms and run the tests. Now
if another hardware platform to the mix, I can just update CSH.
Rarely do I have to update the spec tests themselves.

`CSH.each_hardware` provides a lot of flexibility for test writing.  Say I have a recipe:

~~~ ruby
return if node['virtualization']['role'] == 'guest'

package 'foo'
package 'bar'
~~~

Corresponding tests would be:

~~~ ruby
describe 'dummy::test' do
  CSH.each_hardware do |machine, json|
    context "on #{machine}" do
      let :runner do
        ChefSpec::Runner.new do |node|
          node.automatic.merge! json['automatic']
        end.converge(described_recipe)
      end

      it 'converges' do
        if machine =~ /_Guest$/
          expect(runner.resource_collection).to be_empty
        else
          expect(runner).to install_package('foo')
          expect(runner).to install_package('bar')
        end
      end
    end
  end
end
~~~

When you know a recipe is a no-op, I prefer
`expect(runner.resource_collection).to be_empty` over a bunch of
`expect(runner).not_to` clauses.
