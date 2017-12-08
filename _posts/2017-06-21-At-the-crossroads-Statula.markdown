---
layout: post
title:  "At the crossroads - approaching Statula 0.2.0"
date:   2017-06-21 22:18:11 +0100
---

Hey,
A small update - we are quickly approaching v0.2.0. I plan to roll out few minor updates (v0.1.9
will include few additional starting parameters).   
What is here to come in 0.2.0? io.c will definitely change - I plan to make
read_data more abstract in order to support different notations.  
Let's take a look at a simple dataset:
> 2 2 4 2 2 2 5 4

What if we could represent it as:

> 5x2 2x4 5  

The benefits are not apparent at the first glance, but that is one of many
possibilites.
I will also refactor some of the functions and their names - I would like them
to be as descriptive as possible (and "strings" does not really speak for
itself). 

v0.2.0 will also bring some incompabilities - amongst which is forementioned
abstracted read_data function. Additionally, one of the added command-line
functions will bind statula.c/io.c and the rest together. I would like to avoid
that, but it seems inevitable. I could either stay true to my intentions for
the project (that is - portable library for statistics). 

It is apparent that this will hold the project back in the long run. Some incompabilities were
already introduced back when ___dataset___ structure was added. From v0.2.0 onwards
we steer towards command-line tool. 

Till the next post boys and girls. :)
