---
title: "Painful Ide Switch"
date: 2015-05-28T09:00:22-05:00
draft: false
---

I've been using Eclipse for my Play! + Scala development, but frustration reached the tipping point and I decided to give [IDEA](https://www.jetbrains.com/idea/) a try.  It had been highly recommended by fellow developers at a Scala Meetup (now defunct, sadly).  The conversion process has been painful, with some hard-won lessons to report.

So, what drove me around the bend with Eclipse?  One more bout with it reporting
many false positive compilation errors with no easy way to reset things that I could find.  SBT would give a completely clean compile;  Eclipse would report dozens of errors spread over many, many files.  That, and I can be a sucker for new toys.


## IDEA Installation

These comments are based on IDEA version 14.1.3 and the Scala plugin 1.5.1 back in May, 2015.

The IDEA installation was painful.  Here's what I learned:

1.	**Out of date documentation**:  There is _a lot_ of documentation on the IDEA web site; much of it woefully 
	out of date.  For example, a Google search for "intellij idea scala play" landed me at a 
	features page touting [IDEA 13](https://www.jetbrains.com/idea/features/scala.html)
	with links to a 
	[tutorial](https://confluence.jetbrains.com/display/IntelliJIDEA/Scala) posted 
	in 2013.  As the comments at the bottom of the page atest, it's not applicable.

	If I had landed at this [features overview](https://www.jetbrains.com/idea/features/play_framework.html) instead, things would have been somewhat
	less rocky.

2.	**Ultimate Edition required**:  Working with Play! requires the Ultimate Edition.  This
	tidbit is well hidden.  This [Play! plugin page](https://www.jetbrains.com/idea/features/play_framework.html),
	for example, touts support for templates, routes files, etc.  No where does it mention that the
	Community Edition won't work and the download button at the bottom offers both versions, implying
	that either will work.  The linked 
	[tutorial](https://confluence.jetbrains.com/display/IntelliJIDEA/Play+Framework+2.0) doesn't mention it either.

	It *is* found on the [Scala plugin blog](http://blog.jetbrains.com/scala/2014/09/17/scala-and-play-2-0-plugin-for-intellij-idea-14-eap-is-out/).  It's the last sentence of a paragraph that you need to scroll to see.

3.	**Run Play2 App** may or may not appear in the context menu, as the documentation
	on the [Play! website](https://www.playframework.com/documentation/2.4.x/IDE) and the
	[IntelliJ tutorial](https://confluence.jetbrains.com/download/attachments/49455798/PlayRunningApp.png?version=1&modificationDate=1397129336000) 
	indicate.  I needed to create a run configuration on my own, first:
	1.	Go to "Run --> Edit Configurations..."
	2.	Hit the "+" sign and choose "Play2 App" from the drop-down.
	3.	Give it a name and accept the defaults.

	I still don't find it in the context menu, but it does appear in the Run
	menu.

In retrospect, these are little things.  At the time, however, they were confusing and
chewed up time I could have better spent elsewhere.

## Experience

I'm still getting used to IDEA, but here are some initial observations:

1.	IDEA has a number of code-smell filters ("inspections").  Some of them led to code 
	improvements.  I wish the hover that describes the problem wasn't so sensitive.  It
	disappears quite easily.
2.	There are some false positive type errors -- the same thing that drove me around the 
	bend with Eclipse.  So far they are isolated cases rather than the whole system
	going bonkers as sometimes happened in Eclipse.
3.	I like nested helper functions.  The Eclipse Outline view shows them;  IDEA's 
	Structure view (the closest equivalent) does not.
4.	IDEA detects as soon as I use something that hasn't been imported.  So far it's made 
	good guesses when it offers to automatically add the import.
1.	It seems snappier for editing, but slower than SBT for compiling.
	
