---
layout: post
title: "More hardware-specific testing"
description: ""
category:
tags: ["chefspec"]
---

Most of chefspec examples test whether recipes create various resources.  While
useful, don't overlook that attribute files can also contain logic. This code is
just as worthy of unit tests as recipes.

Say you have a recipe that configures the machine's serial console. Using
`Chef::Util::FileEdit` it manipulates `/etc/inittab`. Based on the hardware, it
could be `ttyS0` or `ttyS1`. And since VMs don't have serial consoles, you don't
want to run `agetty`.

So you've split the logic up between attributes and the recipe. Given the
following attributes file:

~~~ ruby
require 'chef/sugar'

default['console'].tap do |console|
  console['port'] = 'S1'
  console['baud'] = '115200'
  console['term'] = 'ansi'

  unless node.deep_fetch('virtualization', 'role').nil?
    # VMs don't have serial consoles
    console['port'] = nil
  end

  case node.deep_fetch('dmi', 'system', 'product_name')
  when 'PowerEdge M610', 'PowerEdge M710HD', 'PowerEdge M620'
    console['port'] = 'S0'
  end
end
~~~

There are five different paths through this code.  The net result is three
different values for `node['console']['port']`: `S0`, `S1`, and `nil`.

~~~ ruby
describe 'console::default' do
  CSH.each_hardware do |desc, ohai|
    context "on #{desc}" do
      let :runner do
        ChefSpec::Runner.new do |node|
          node.automatic.merge! ohai['automatic']
        end.converge(described_recipe)
      end

      it 'converges' do
        case desc
        when /_Guest$/
          expect(runner.node['console']['port']).to be_nil
        when /^PowerEdge_(M610|M710HD|M620)$/
          expect(runner.node['console']['port']).to eq('S0')
        else
          expect(runner.node['console']['port']).to eq('S1')
        end

        expect(runner.node['console']['baud']).to eq('115200')
        expect(runner.node['console']['term']).to eq('ansi')
      end
    end
  end
end
~~~

Now that the attributes have been verified, you can write spec tests for the recipe
code.
