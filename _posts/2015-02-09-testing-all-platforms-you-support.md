---
layout: post
title: "Testing all platforms you support"
description: ""
category:
tags: ["chefspec"]
---

A few days ago, I blogged about using `CSH.each_hardware` to ensure you have
code coverage for the various hardware platforms you support. I use the same
methodology for testing all the OS/version combinations we support.  In the CSH
(ChefSpecHelper) gem the following code resides:

~~~ ruby
module CSH
  PLATFORMS = {
    supported: [
      %w[centos 5.8],
      %w[centos 6.3],
      %w[centos 6.4],
      %w[centos 6.5]
    ],
    unsupported: [
      %w[ubuntu 12.04]
    ]
  }

  def self.each_platform(*kinds, &block)
    array = []
    kinds = [:supported] if kinds.empty?
    kinds.each do |k|
      fail "Unknown kind #{k.inspect}" unless PLATFORMS.include?(k)
      array.concat PLATFORMS[k]
    end
    array.map do |platform, version|
      yield platform, version
    end
  end
end
~~~

Say you have a dummy::grub recipe that is platform specific:

~~~ ruby
package 'grubby' do
  package_name 'mkinitrd' if node['platform_version'].to_i < 6
end
~~~

The corresponding spec test would look like:

~~~ ruby
describe 'dummy::grub' do
  CSH.each_platform do |platform, version|
    context "on #{platform}-#{version}" do
      let :runner do
        ChefSpec::Runner.new(platform: platform, version: version).converge(described_recipe)
      end

      it 'converges' do
        expect(runner).to install_package('grubby')
        if version.to_i < 6
          expect(runner).to install_package('grubby').with(package_name: 'mkinitrd')
        end
      end
    end
  end
end
~~~

The great thing about controlling this centrally is that you don't
have to modify spec tests across your complete cookbook set. In the
next weeks when we power off the remaining CentOS-6.3 machines, I'll
just remove the entry and we'll stop executing those tests.

You can combine this methodology with `CSH.each_hardware` and ensure that
all supported nodes are covered.

~~~ ruby
describe 'raid::utils' do
  CSH.each_platform do |platform, version|
    CSH.each_hardware do |machine, json|
      context "#{platform}-#{version} on #{machine}" do
        let :runner do
          ChefSpec::Runner.new(platform: platform, version: version) do |node|
            node.automatic.merge! json['automatic']
          end.converge(described_recipe)
        end

        it 'converges' do
          if machine =~ /^PowerEdge/ && version.to_i == 5
            expect(runner).to install_package('raidcfg')
          else
            expect(runner).not_to install_package('raidcfg')
          end

          if machine =~ /^PowerEdge/ && version.to_i == 6
            expect(runner).to install_package('MegaCli')
            expect(runner).to install_package('MegaLogR')
          else
            expect(runner).not_to install_package('MegaCli')
            expect(runner).not_to install_package('MegaLogR')
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
end
~~~

Sure you could collapse these tests down into just the bare minimum
needed to have coverage.  But why?  Since chefspec is so fast, you
might as well take a more black box approach to these tests. In the
future there may be more or different permutations to test. It is
better to have the scaffolding in place instead of having to retrofit
it later.
