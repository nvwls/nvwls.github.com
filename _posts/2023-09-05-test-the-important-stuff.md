---
layout: post
title: "Test the Important Stuff"
description: ""
category:
---

I am a huge proponent of writing as little code as necessary to do the job.
This applies to tests as well. Tests are just code. In code reviews I often
tell people to "just test the important stuff". Sure you can test
_everything_ but what is purpose in that. The more code, the higher the
maintenance cost. The same is true of tests.

Let's configure the [Turbo Encabular](https://youtu.be/Ac7G7xOG2Ag):

```ruby
cookbook_file '/etc/sysconfig/turbo-encabulator' do
  action :create
  owner 'root'
  group 'root'
  mode '0644'
end
```

And write an inspec test for it:

```ruby
describe file('/etc/sysconfig/turbo-encabulator') do
  it { should exist }
  its('owner') { should eq 'root' }
  its('group') { should eq 'root' }
  its('mode') { should cmp '0644' }
end
```

While the test is good, it is just a direct translation of the recipe. It
could, in theory, be mechanically generated. Your tests have just become:
did chef do what I specified? At this point you're almost writing regression
tests for chef-client instead of your recipe code.

Let's look at the mode on the file: `0644` that means the owner can write and
everyone else can read. Since it is `g+r` and `o+r`, the group is
unimportant. If the group does not matter, why test it? The `mode` test is
more useful. It would catch if the file was `0666` for example.

So the most important things are:

1. The file exists
2. The mode is `0644` so others cannot write it
3. The owner is `root` so only someone with sudo privileges can modify it

Since the group has no bearing, why even test it? The inspec test now
becomes:

```ruby
describe file('/etc/sysconfig/turbo-encabulator') do
  it { should exist }
  its('owner') { should eq 'root' }
  its('mode') { should cmp '0644' }
end
```

One could argue that the `exist` test is unnecessary: if the file does
not exist, the `owner` and `mode` tests would fail. This is true.

```output
  File /etc/sysconfig/turbo-encabulator
     ×  owner is expected to eq "root"

     expected: "root"
          got: nil

     (compared using ==)

     ×  mode is expected to cmp == "0644"

     expected: 0644
          got:

     (compared using `cmp` matcher)
```

However, I believe that the additional check makes reading the output easier.

```output
  File /etc/sysconfig/turbo-encabulator
     ×  is expected to exist
     expected File /etc/sysconfig/turbo-encabulator to exist
     ×  owner is expected to eq "root"

     expected: "root"
          got: nil

     (compared using ==)

     ×  mode is expected to cmp == "0644"

     expected: 0644
          got:

     (compared using `cmp` matcher)
```

When I read it, I see the `is expected to exist` failure and can stop
reading. If the file doesn't exist, then none of the subsequent tests could
possibility be true. I don't have to remember that `owner` will be nil if
the file does not exist.

Let's modify the original recipe. Say there was some piece of sensitive
information in the file so you restrict permissions on the file.

```ruby
cookbook_file '/etc/sysconfig/turbo-encabulator' do
  action :create
  owner 'root'
  group 'wheel'
  mode '0440'
end
```

Now, the group *DOES* matter so the test should reflect that:

```ruby
describe file('/etc/sysconfig/turbo-encabulator') do
  it { should exist }
  its('owner') { should eq 'root' }
  its('group') { should eq 'wheel' }
  its('mode') { should cmp '0440' }
end
```

While this may seem like trite example, it is a concise example. The
argument really makes shines when you test the contents of a file.

```ruby
describe sshd_config do
  its('PermitRootLogin') { should eq 'no' }
end
```

Disabling root login is important, so you should test that. If your recipe
does not override `PidFile` is it that important to test?

```ruby
describe sshd_config do
  its('PidFile') { should eq '/var/run/sshd.pid' }
end
```

You could write tests for each and every sshd config option but what value
does that add? Concentrate on the "important" pieces.

The more code there is, the perceived complexity of it is higher. It can be
daunting entering a new code base with thousands of lines. You think that
you need to understand all the code before you can make a change. But if
the code base is only 100 lines, it seems more manageable.
