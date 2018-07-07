+++
author = "admin"
categories = [".NET", "algorithm", "C#"]
date = 2009-10-25T20:11:07Z
description = ""
draft = false
slug = "dijkstras-algorithm-in-c"
tags = [".NET", "algorithm", "C#"]
title = "Dijkstra's algorithm in C#"

+++


I had the pleasure of working with this algorithm one year ago during a project on route planning. Recently I encountered this algorithm again during a course, and decided to do a small implementation of this algorithm along with a graphical interface to use it. Use it for anything you want.

It finds the shortest path from a starting point to all nodes in a graph, but keeps track of how to get there, so if you want to use it for the shortest path between 2 points, simply follow the previous node located in *path *from the destination point.

**04-06-2012 update: **Despite this post being nearly three years old it still generates a lot of traffic and interest in the download. I would like to state that the implementation is likewise somewhat outdated and uses old language constructs and the like. I recommend it to be used as a guide and inspiration.

[http://files.nocture.dk/pub/Dijkstra.zip](http://files.nocture.dk/pub/Dijkstra.zip)


