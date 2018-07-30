---
title: "When Naming Is Important, and When It's Not"
draft: false
date: "2018-07-30 09:17:00"
categories: 
  - Development
tags: 
  - Development
  - Standards
  - Programming
keywords:
  - go
  - golang
  - development
  - best practices
  - gotchas
  - naming
  - naming standards
  - programming
---

# Like Neanderthals Grunting and Pointing at Cave Paintings
The other day a junior dev walked up to my desk and ask for some help troubleshooting a deadlocking issue he was seeing.
I wasn't familiar with the details of his process, so he proceeded to explain to me how the data was modeled, and how it
flowed from one process to the next and was modified along the way.

The process was complicated enough that I wasn't able to build a clear picture of the process in my head.  I found I was
beginning to misunderstand some of the nuances of the process while he was explaining it.  After a bit, when we both 
realized we weren't on the same page yet, we decided to quick sketch out the process on a whiteboard.

Now, I'm not the most artistic individual.  In fact, even saying that I'm "not artistic" is being *so* generous as to be
absurd.  In truth, I couldn't draw a straight line to save life, and even when I'm writing notes down on paper (instead
of on a keyboard), I find myself getting impatient with the speed at which I can get ideas down, and frustrated with
the fact that I can barely read what I've just written.  

All of this goes to the point that when I whiteboard something it's more than a bit of a mess.  It's a collection of 
wild lines connecting oblong circles, and lopsided squares with random chicken scratches scattered all over that could 
be anything from abbreviations to Entity-Relationship markings. Or maybe it was just where I accidentally leaned on the 
board for a minute.

Despite all of my dexterous inadequacies, I usually manage to get my ideas across, especially if I force myself to slow
down and be patient when whiteboarding an idea.  This *wasn't* one of those times.

I ended up being able to help him out, and we planned out a better way to model some of the data in his process so that
it would perform better concurrently.  After he left, I went over to clean up the whiteboard.  I was expecting the mess
of a quickly sketched out idea, but what was left on the board could only be described as *"abstract"*.   There where
random lines darting back and forth from one shape to the next.  There seemed to be no order or reason to any of the 
"shapes" on the board.  Once I had stepped away from the problem, and had let the *context of the problem* drop out of 
my current working set of ideas, there was nothing left but chaos.  And most surprisingly, ***there wasn't a single word
or character anywhere***.

# Shared Context
I thought of the fervent back and forth of our discussion.  How did we manage to communicate complicated ideas 
so effectively?  Our first attempt without the whiteboard was a failure, but somehow adding the whiteboard into the 
discussion completely facilitated effective communication.  Not only that, but we did it without having to *name* any of 
the myriad, individual parts of the process.  There were no "DataProcessFactory Factories" or "AbstractManagerImpl".  

The whiteboard, even though messy and disorganized, provided *shared context*.  When I referred to the piece of code
that distributed data to several child processes, I drew a square (or at least a roughly square-like object).  From that 
moment on, that square(ish) shape encapsulated the idea of that process.  Every time I, or the junior dev, needed to refer
to that process we simply pointed (and grunted).  It didn't need a name, because we shared a **Contextual and Agreed Upon
Plane of Definitions** or **CAUPD**.  On second thought, that name sucks.  We'll just call it shared context.

# Naming Things is Hard
So how does this apply to programming?  What great insight did this this all lead me to, and how did it make me a better
developer?  Well it didn't really.  I'm not doing anything different today that I wasn't already doing.  However, it has 
given me a better understanding as to the *why* of some naming and programming standards that I keep seeing being brought
up again and again when best practices are being discussed.

Names are a way to share context.  Either with another developer, or with your idiot self in six months when you've
forgotten how any of your code works.  The name needs to carry along the entire concept of what that code is doing.  Oh,
and it can't be too long because no one likes long names.  And you need to come up with the name on the spot while writing
your code, and all while you might not know entirely everything that the code will need to do because the client hasn't
given you all of their requirements yet.

Naming things is hard.

# Shrink Your Context
When I stepped away from that whiteboard I *lost the context of the problem*.  It was at that point the scribbles
and smudges of my diagram changed from a **Glorious Diagram of Identifiers and Logic** to an unintelligible mess.  It 
turns out that a misshapen square isn't a good substitute for a proper name.  With that in mind, you need to expect that
anyone looking at your code is coming at it with *no* context.

The best way to help someone understand your context is to make that context as small as possible.  When is `i` a better
variable name than `StudentRecordIndex`?  When `i` lives and dies in the handful of lines before and after it. When they
can quickly forget what `i` means because it's no longer relevant to the rest of the code.

If you keep your functions / methods / classes small and focused on doing a specific thing, your variables will name
themselves. You can then get away with naming your variables with a single letter; the development equivalent of pointing 
and grunting.