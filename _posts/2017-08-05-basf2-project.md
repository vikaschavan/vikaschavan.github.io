---
title: "basf2 - Belle II Software and Analysis Framework"
layout: post
date: 2017-08-05 10:00
tag: [python, C++, physics]
image: "http://wporpt.c.blog.so-net.ne.jp/_images/blog/_48a/wporpt/20120902-01.jpg"
headerImage: true
projects: true
hidden: true
description: "basf2 - Belle II Software and Analysis Framework"
category: project
author: nils
externalLink: false
---

Belle II is the successor of - you might have guessed it - the Belle experiment and is located at the SuperKEKB
facility in Tsukuba, Japan. It is a so called "B-factory".
Its name stems from the fact, that it "produces" B-mesons.
So, what are B-mesons and why would someone wants to produce them?

![A Proton](https://upload.wikimedia.org/wikipedia/commons/thumb/9/92/Quark_structure_proton.svg/1200px-Quark_structure_proton.svg.png){:class="center-image"}
*A proton with its quark content*

The nature around you as you know it consists of molecules, which consist of atoms, which consist of electrons and protons
and neutrons (this is what they tell you at school :smile:). Protons and neutrons are however not the end of the
story, but there is still another subpart: they consist of "quarks". There are 6 different types of quarks, and proton
and neutron consist of two of them (the "up" and the "down" quark). Number 5 in the family of quarks is the "bottom" or
"beauty" quark - and objects build with b-quarks are called B-mesons (this is not the full story, but I want to keep
it simple). B-mesons are very interesting, as they could have properties, which are directly related to new,
unknown physics and may solve very important problems, as dark matter or the origin of the universe.

The only problem is: there are no B-mesons in our world! Why? Well, they are so short-lived, that they would decay
immediately if they would be produced somehow in our world. So we have to generate them somehow on our own.
We do this, by shooting energetic beams of electrons and positrons onto each other at a very specific energy. As we need
many of those B-mesons to investigate on their properties, we do this every 20 ns (this is very fast!) and during our
measurements period of a few years, we end up with billions of recorded B-meson pairs.

The B-mesons that appear in these high energetic collisions are so short-lived, that they decay more
or less instantaneously int other so called final state particles. One of them is the electron (or its
partner the positron). As we want to know what happened during these initial collision, we measure as many properties
of the produced final state particles as we can: their charge, their momentum, their energy and their type. We do this
by bending them in a large and strong magnetic field with a force,  called the Lorentz Force (link). Their trajectory
can then be measured with semi-conductor sensors (something like a large digital camera) or chambers full of gas,
which gets ionized by the charged particles (think about your neon lamp, which works with a similar mechanism).
As we ant to know their trajectory very precisely, we have installed millions of these sensors in our detector, which
has the size of a small house.

[![Belle II Detector](https://i.ytimg.com/vi/nGCrrgXSEOk/maxresdefault.jpg)](https://www.youtube.com/watch?v=nGCrrgXSEOk){:class="center-image"}
*The Belle II detector (youtube link <https://www.youtube.com/watch?v=nGCrrgXSEOk>)*

From the radius of the trajectory, we derive the charge and the momentum and with some
more data, also their energy, their mass and their type.

The problem-to-solve is then: how to find circles or arc segments in this large amount of sensor measurements in each
event - and do this with very tight time requirements? This is what we call "tracking" - and it is one of the most
important steps in reconstructing the whole event. We use algorithms known from pattern recognition in images for this talk,
and multivariate methods do distinguish between sensor measurements of physical interesting processes and so called
background, which stems from uninteresting side products of the collision and the particle beams.

[![A simple tracking algorithm](https://pbs.twimg.com/media/C6zUZBuWwAYDm5n.png)](https://twitter.com/belle2collab/status/841280775651807238){:class="center-image"}
*The final result of a short hacking challange, on last years [CTD conference](https://twitter.com/belle2collab/status/841280775651807238)*

We use C++ for our algorithms and Python for controlling their parameters and analyzing them (because we like Python!).
Our framework with multiprocessing, C++-object-streaming, the in- and output mechanisms and the different reconstruction
algorithms is quite advanced and has way more than 100.000 lines of code.

Besides the work on such a huge project, which is interesting on its own, you get in contact with many interesting
technologies, e.g. as we need a continuous integration system (where we use virtual machines), a way to store all
this data reliably, an issue tracker, a wiki (where we use the atlassian stack) etc.
And you make contact with very smart people from all over the world!