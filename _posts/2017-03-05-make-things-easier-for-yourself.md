---
layout: post
title: "Make Things Easier for Yourself"
description: ""
category:
tags: []
---

In the early 80s I remember going with my dad to have the oil changed on his
company car. The common practice at the time was to put the current date and
mileage on the window sticker. My dad was adamant that they put the date as +3
months and the mileage as +3000 miles. He told them that it was easier because
he didn't have to do the math every time he checked the odometer to see if he
needed to get an oil change. He drove a lot for his job so he never had to worry
about the date.

When I started driving, it was common practice for the oil change window
stickers to have the "due" date instead of the "done" date.  Was the shift in
the industry due to my dad? Probably not. It is moot point nowadays because
modern cars tell you when the oil needs to be changed.

That whole concept of "make things easier for yourself" always stuck with me.

In woodworking, [jigs](https://en.wikipedia.org/wiki/Jig_(tool)) are super
important. In my youth I used to help my dad built cabinets for his side
business. We would often spend a lot time building the jig and then zip though
the build. What I didn't understand at the time was that we were able to zip the
build because the jig saved us so much time. We didn't to stop and measure, or
really think about it.

Through my career in software engineering I've used automation. I've used (and
abused) [make](https://en.wikipedia.org/wiki/Make_(software)) and
[rake](https://en.wikipedia.org/wiki/Rake_(software)) over the years. I've
written (and rewritten) an ssh-wrapper to deal with multi-hop environments. To
paraphrase Barbara Mandrell, I was
[automating before automating was cool](https://en.wikipedia.org/wiki/I_Was_Country_When_Country_Wasn't_Cool).

But automation is just part of it. Just as my dad didn't want to do math
every time, I've taken that approach as well with alerts. Instead of being alerted
that disk is 95% full, I prefer to get notified with "less than 10G free".

Another common alert is for expired certificates. People usually raise a warning
some days beforehand.  Sure you can calculate dates in your head but why? Let
computers do that for you. Factor in lead time. If you know it takes 3 days to
get a certificate renewed, then make sure the warning takes that into account.

And while you're at it, factor in regular business hours as well. Sure, "10 days
beforehand" is easy to code but it might mean the alert fires at 02:15 on
Saturday. Since you're most likely not going to be able to do anything about it
until Monday, why not defer the alert until then?

I built a simple tool that based on a timestamp, it would calculate "09:00 Monday
at least 7 days prior".

Just as jigs used in woodworking, it'll take sometime to build these tools. It
may not seem worth it at the time but it is hard to quantify not getting paged
over the weekend for benign issues. What will you do with that time? I don't
know about you but it looks like I need to take the car in for an oil change.
