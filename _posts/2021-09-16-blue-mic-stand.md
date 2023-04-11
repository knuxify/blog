---
layout: post
author: knuxify
title: "Fixing the Blue Snowball microphone stand, with the power of 3D printing"
image: /images/blue-mic-stand/micstand1.jpg
tags:
  - 3d printing
  - 3d
  - stl
  - fix blue snowball stand
category: Projects
---

I have a Blue Snowball microphone. As it usually is with any items that end up on my desk, I'll inevitably end up fidgeting with them in one way or another and breaking them as a result. And the same rule applied in this case - I broke the mic's stand fairly quickly. (Although, to be fair, it was fairly weak to begin with).

What happened? Well, the legs just started flapping about, and when you moved the mic too much the little "bolts" holding them on would fall out. This wasn't all too bad, considering that the mic could still stand on its own, but eventually my fidgety hands managed to make one of the "bolts" fall out, never to be seen again. (I also lost the third leg that was being held in place by said bolt.)

I knew that getting another original stand would just postpone the issue's re-appearance, so I turned to the only other method of making my own stand that I could find - 3D printing.

<!--more-->

## Finding the model

<figure>
	<img alt="My 3D printer, an XYZprinting Da Vinci mini w (ignore the snapped filament)." src="{{ site.baseurl }}/images/blue-mic-stand/printer.jpg">
	<figcaption>My 3D printer, an XYZprinting Da Vinci mini w (ignore the snapped filament).</figcaption>
</figure>

Admittedly, I haven't really had an excuse to use it for quite a while, but I figured I'd give it a shot.

My first thought was to simply 3D-print a small stand from a model from Thingiverse. I found [a simple model](https://www.thingiverse.com/thing:4685978), and decided to print it out.

Only one problem, though - the screw ended up snapping soon after I started using it.

I'm assuming setting higher infill settings would make it sturdier, but I didn't really think about testing that out back then. Besides, this isn't necessarily the best printer, and my filament is already a few years old, both factors which could've increased the likelyhood of my prints getting bad.

I then stumbled upon [a blog post about yet another 3D-printable stand design](https://willj.net/posts/custom-blue-snowball-microphone-desk-stand/), and decided to give it a shot too. However, the thin stem broke instantly when I mounted the microphone on the stand. I tried modifying this model to make the screw stand flat on the base, but the screw ended up giving up soon enough as well.

## The solution

Then it hit me. The screw is the weakest part of all of my prior print attempts... but the screw in the original stand is still intact!

In case you didn't know: in the original stand, the bolts and legs are held in by an unscrewable plastic bottom cover. I figured that I could simply replace that bottom cover with a 3D-printed replacement part.

I quickly glued something together: I took the base stand part from Will's model (from the blog post I linked earlier), and took the bottom cover from [Thingiverse](https://www.thingiverse.com/thing:3870129), and here's the result:

<figure>
	<img alt="The fixed stand, with a 3D-printed bottom cover." src="{{ site.baseurl }}/images/blue-mic-stand/micstand1.jpg">
	<figcaption>The fixed stand, with a 3D-printed bottom cover.</figcaption>
</figure>

It's not perfect; the bottom cover model I found doesn't have the slight groove required for it to sit flush with the top of the microphone; however, this doesn't prevent it from being attached sturdily.

<figure>
	<img alt="The underside of the 3D-printed bottom cover." src="{{ site.baseurl }}/images/blue-mic-stand/micstand2.jpg">
	<figcaption>The underside of the 3D-printed bottom cover.</figcaption>
</figure>

The 3D-printed cover simply screws in from the bottom using the original screw. You can see that my printer struggled with the curved underside (I wanted to add supports, but the buggy proprietary software for the printer didn't want to cooperate...), but other than that, the print came out fairly well, and I'm satisfied with the result!

If you want to print your own, you can find the model under this link: [Blue Snowball stand fix](https://files.dithernet.org/blue-snowball-stand-fix) / [Thingiverse mirror](https://www.thingiverse.com/thing:4966494).
