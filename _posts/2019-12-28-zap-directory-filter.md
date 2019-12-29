---
layout: post
title: "zap_directory now supports filter"
description: ""
category:
tags: ["chef", "zap"]
---

I'm happy to announce v1.2.0 of the [zap cookbook](https://github.com/nvwls/zap).

The new feature is that `zap_directory` now supports `filter` similar to `zap_users` and
`zap_groups`.

As Plato said "necessity is the mother of invention", this came from the need to zap files from
`/etc/modprobe.d`. The first pass of this was:

~~~ ruby
zap_directory '/etc/modprobe.d' do
  register :modprobe
end
~~~

`modprobe` is a simple custom resource that wraps a template. This zap'ed the resource I commented
out for testing as I expected. But it also zap'ed `/etc/modprobe.d/lockd.conf` which comes with the
nfs-utils RPM.

So I wanted to make sure that files delivered by RPMs don't get removed.  I could override `collect`
but it seemed that `filter` was the cleaner approach.

~~~ ruby
# Remove files from /etc/modprode.d that we don't know about
zap_directory '/etc/modprobe.d' do
  register :modprobe

  filter do |path|
    # Keep vendor provided config files
    !shell_out("rpm -qf #{path}").status.success?
  end
end
~~~

`collect` will accumulate paths that `filter` returns true. Since `rpm -qf` returns success if the
path is part of an RPM, we only want to accumulate paths that are NOT part of an RPM. So we need to
negate the result to get the desired behavior.

Happy chef'ing!
