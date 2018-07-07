+++
author = "admin"
categories = ["Ableton", "Music"]
date = 2009-11-11T20:13:52Z
description = ""
draft = false
slug = "ableton-import-error-could-not-be-read-it-may-be-corrupt-or-not-licensed"
tags = ["Ableton", "Music"]
title = "Ableton import error \"Could not be read. It may be corrupt or not licensed\""

+++


*Here is a possible fix for you if you encountered this problem when trying to import some of your music files into Ableton.*

Finally I had some spare time to mess with some music again, but encountered a problem when using Ableton that I had not experienced before. After trying to import some of my music clips in Mp3 format, the error message “Could not be read. It may be corrupt or not licensed” appeared, even though I had done the same plenty of times before.  
 I decided to do some digging, and plenty of people seem to have encountered this issue, but with plenty of suggestions of ways to solve it, that did not help me in my case. I then noticed that my mixing software “Traktor” had written some custom tags to my soundfiles if they had been analyzed, after reading a forum post of Traktor causing this for some reason. For some reason Ableton consider these soundfiles corrupt when encountering such custom tags.

If you use Traktor/any other possible music software as well as Ableton and encounter this issue, remove all custom tags from your soundfiles and it should work (unless the file is actually corrupted or DRM’ed ![:)](https://nocture.dk/wp-includes/images/smilies/icon_smile.gif) ).  
 My favourite audioplayer [Foobar2000](http://www.foobar2000.org/) is able to do this easy, simply right click the soundfile(s) and choose properties, and then select remove tags from the tools button (do this under both Metadata and Properties tabs).

Happy producing ![:-)](https://nocture.dk/wp-includes/images/smilies/icon_smile.gif)


