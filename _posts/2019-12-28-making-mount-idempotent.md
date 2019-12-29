---
layout: post
title: "Making mount idempotent"
description: ""
category:
tags: ["chef"]
---

Recently ran into an interesting issue with a tmpfs mount.

Originally, I had created a 1G tmpfs and the recipe code looked like:
~~~ ruby
size = '1G'

mount '/dev/shm' do
  device     'tmpfs'
  fstype     'tmpfs'
  options    ['rw', "size=#{size}"]
  action     [:enable, :mount]
end
~~~

Later it was bumped to 2G. When `chef-client` ran, only `enable` action was performed.

~~~
Recipe: base::_tmpfs
  * mount[/dev/shm] action enable
    - enable tmpfs
  * mount[/dev/shm] action mount (up to date)
~~~

`/etc/fstab` was updated:
~~~
# grep /dev/shm /etc/fstab
tmpfs /dev/shm tmpfs rw,size=2G 0 2
~~~

But the mount was not:
~~~
# mount | grep /dev/shm
tmpfs on /dev/shm type tmpfs (rw,relatime,size=1048576k)
~~~

So `:mount` just ensures it is mounted, it does not verify options.  There is a `:remount` action.
Let's try that:

~~~
Recipe: base::_tmpfs
  * mount[/dev/shm] action enable
    - enable tmpfs
  * mount[/dev/shm] action remount
    - unmount tmpfs
    - mount tmpfs
~~~

Looks good:
~~~
# grep /dev/shm /etc/fstab
tmpfs /dev/shm tmpfs rw,size=2G 0 2

# mount | grep /dev/shm
tmpfs on /dev/shm type tmpfs (rw,relatime,size=2097152k)
~~~

But it did `:unmount` followed by `:mount`.  Oh, I need to specify that `:remount` is supported.

~~~ ruby
mount '/dev/shm' do
  device     'tmpfs'
  fstype     'tmpfs'
  options    ['rw', "size=#{size}"]
  supports   :remount => true
  action     [:enable, :remount]
end
~~~

And ran it:
~~~
Recipe: base::_tmpfs
  * mount[/dev/shm] action enable
    - enable tmpfs
  * mount[/dev/shm] action remount
    - remount tmpfs
~~~

So it just did a `:remount`, good! But on subsequent runs it did `:remount` again:
~~~
Recipe: wd_base::_tmpfs
  * mount[/dev/shm] action enable (up to date)
  * mount[/dev/shm] action remount
    - remount tmpfs
~~~

Looking at the mount:
~~~
# mount | grep /dev/shm
tmpfs on /dev/shm type tmpfs (rw,relatime,size=2097152k)
~~~

Ah, maybe it is `relatime` which is the default. Also, size is specified in k not G.
~~~ ruby
size = "#{2 * 1024 * 1024}k"

mount '/dev/shm' do
  device     'tmpfs'
  fstype     'tmpfs'
  options    ['rw', 'relatime', "size=#{size}"]
  supports   :remount => true
  action     [:enable, :remount]
end
~~~

The options match but it is still doing `:remount`!
~~~
Recipe: base::_tmpfs
  * mount[/dev/shm] action enable (up to date)
  * mount[/dev/shm] action remount
    - remount tmpfs
~~~

This still not idempotent. Ohai to the rescue. I reworked the recipe a bit:
~~~ ruby
size = "#{2 * 1024 * 1024}k"

mount '/dev/shm' do
  device     'tmpfs'
  fstype     'tmpfs'
  options    ['rw', 'relatime', "size=#{size}"]
  supports   :remount => true
  action     :enable

  have = node.read('filesystem2', 'by_mountpoint', name, 'mount_options')
  if have.nil?
    # not mounted, mount
    action << :mount
  elsif have != options
    # mount options differ, remount
    action << :remount
  end
end
~~~

When the options are the same, only `:enable` is performed.
~~~
Recipe: base::_tmpfs
  * mount[/dev/shm] action enable (up to date)
~~~

When the filesystem is not mounted, it will `:mount` because have is nil:
~~~
Recipe: base::_tmpfs
  * mount[/dev/shm] action enable (up to date)
  * mount[/dev/shm] action mount
    - mount tmpfs to /dev/shm
~~~

And if the options don't match, it will `:remount`:
~~~
Recipe: wd_base::_tmpfs
  * mount[/dev/shm] action enable (up to date)
  * mount[/dev/shm] action remount
    - remount tmpfs
~~~

As you can see, sometime chef needs a little help in being idempotent.

Happy chef'ing!
