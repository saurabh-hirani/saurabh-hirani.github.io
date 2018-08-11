---
layout: post
title: Creating custom watchfaces for Amazfit Bip
tags:
- amazfit
---

[Amazfit Bip](https://www.techradar.com/sg/reviews/amazfit-bip) is a simple, no-nonsense
smartwatch. For a user with basic needs like me - (a clean digital face with step counter),
it serves the purpose really well. I liked one of the following stock digital watch faces:

{% include figure.html path="blog/amazfit-watchface/amazfit-default-watchface.png" alt="Amazfit default watchface" url=url_with_ref %}

But there were some glaring limitations in it that I wanted to fix:

1. It has a seconds display - which is a drain on the battery because the display refreshes every second.
2. There is no way of knowing how much battery is left - the current indication might be for 15, 20, 30%.
3. There are no icons for - bluetooth, alarm, dnd or watch lock status.

I browsed several watch faces on the [Amazfit watch faces site](https://amazfitwatchfaces.com/bip/) in the
hope that someone might have noticed the same things as I did, and hopefully fixed them. But I couldn't find
any fixes for the above.

I ranted about the same on [reddit](https://www.reddit.com/r/amazfit/comments/965b3v/is_anyone_using_a_casio_style_progress_meter_with/)
and a user was kind enough to give me a basic fix which displayed battery amount, bluetooth and alarm status. But it still showed 
seconds and the battery amount font was still small.

I poked around a bit and got the instructions to modify watchfaces on [this thread](https://www.reddit.com/r/amazfit/comments/83q6wl/watchface_coding/).
It just required some basic image editing and json config changes.

Did that and I released 2 versions of my work:

1. [Digital progress meter with icons](https://amazfitwatchfaces.com/bip/view/?id=11283) 
   - [Github repo for the above .bin file](https://github.com/saurabh-hirani/amazfit-watchface-digital-progess-meter)
2. [Digital progress meter with icons v2](https://amazfitwatchfaces.com/bip/view/?id=11305)
   - [Github repo for the above .bin file](https://github.com/saurabh-hirani/amazfit-watchface-digital-progress-meter-v2)

This is a gif of the v2 watchface that I created:

{% include figure.html path="blog/amazfit-watchface/amazfit-modified-watchface.gif" alt="Amazfit default watchface" url=url_with_ref %}

Lesson learnt: 

- I never thought I would do need to do image editing or simple design work. But we learn everything we need to when we want to scratch our own itch.
