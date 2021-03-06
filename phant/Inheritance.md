---
title: Inheritance in the Kernellib
layout: default
---

The main limitation the Kernel MUDLib imposes on inheritance is
the division of objects into inheritables, clonables, and objects
from which LWOs can be created

They're primarily distinguished from each other by what
directory the files are in -- if the path contains a "/lib/" then
the program is an inheritable. Inheritables may not be cloned, but
may be inherited from other programs. They can't store any data,
you can't find them with find_object(), and no functions (including
create()) are ever called on them. The Kernel Library specifically
prevents you from touching the 'real' object for reasons that will
be explained later.

If a path contains a "/data/" then the object is a master for
LightWeight Objects (LWOs) and can be instantiated with
new_object(), but may not be cloned or inherited from. Note that a
file with "/data/" in the path may inherit from inheritables just
like everybody else. It also has data, and you can call functions
on it. You can also call functions on LWOs that you make from it,
and (of course) they can have their own copies of any data.

An object whose path contains "/obj/" is a cloneable. You can
use clone_object() to make clones of it, but other programs can't
inherit from it and you can't call new_object() on it to make an
LWO from it. You can call functions on it (the master object
<i>and</i> any of the clones), and it has usable data.

Any other path (one without either "/data/" or "/lib/" in the
path) is assumed to be nothing special. You can't clone it, you
can't make an LWO from it, you can't inherit from it. You can find
it (remember, there's only the one master object) with
find_object(), you can call functions on it and it can have data.
Most daemons are this way in Kernel-derived MUDLibs. Usually
authors will put "/sys/" in the path instead of "/obj/" or "/lib/"
as documentation... But it's not required.

Okay, so why can't inheritables have data, or be cloned? Seems
like a pretty serious limitation if you think about it a bit. It
means that every parent class is an abstract parent class rather
than being instantiable (to use some fancy Object Oriented
terminology). The answer has everything to do with the way DGD
allows you to recompile everything on the fly.

You can recompile an object with clones and at the end of that
thread, all the clones get upgraded. It works for LWOs, too. That's
pretty cool. You can recompile a library, and from then on any new
objects that inherit it get the new version. Also cool.
Unfortunately, old objects can't just switch which version of the
parent class's code they use, so they're stuck with the old version
of the library until you recompile them (after destructing and
recompiling the library).

In the Kernel MUDLib you can deal with that -- just destruct the
old version and recompile the clonable (see Issues, below, for an
example). That'll upgrade all the clones, give you the library, and
everything stays copacetic. Since a library has no clones and no
data, when you destruct it and recompile, you lose nothing. Since
the clonable has nothing inherit from it, you don't need to
destruct it and recompile for the benefit of <i>its</i> child
classes (since it has none).

So what if you <i>don't</i> do that? Melville and 2.4.5 both get
away without doing any of this. The answer is that you can't
upgrade some objects when the MUD is running, so you can kiss
full-on persistence goodbye. To understand why, think about what an
object which is both inheritable <i>and</i> clonable would be like
to upgrade. In OO-speak, that's a concrete (non-abstract) parent
class.

If the object were both inheritable and clonable then it could
have some objects that inherited from it, plus a bunch of clones.
If you wanted to upgrade it, you'd need to be able to destruct it
and compile a new one so that the classes that inherit from it
could get a new version. But that's a problem... You'd have to lose
all the data in all the clones when you destructed it! So it can be
both upgradable and clonable, but you couldn't upgrade its
functions in its child classes without getting rid of all the
clones... You can see why the Kernel Library just separates
inheritables and clonables.

Could you make a different tradeoff to avoid having both child
objects and clones? Probably. For instance, you could make an
object be both inheritable and clonable, but make it destroy all
its clones to upgrade. That'd be a massive pain, but you could do
it. There are a lot of compromises like that, but the Kernel
MUDLib's is simple, reliable and it works. If you want a different
one, you can write a different MUDLib (or modify the Kernel MUDLib)
to show us all how much better the world would be with your
version...

With great power comes great responsibility, to quote an old
comic book character you've all heard of. So the Kernel MUDLib
makes a tradeoff -- you can recompile everything in the MUD if you
want to, but in return you have this set of restrictions.

<hr>

## Recompilation and Object Issues

Each time an object is destructed and then compiled again from
scratch, a new copy of it shows up. The old copy will silently go
away when nobody's using it any more. This means that old versions
of objects can float around for a very, very long time (remember
that "Persistent MUD" idea?) in some cases.

How can you tell how it works? Well, when no existing object
uses an old issue, it goes away. This uses Reference Counting, so
when the reference count drops to zero, DGD knows that nobody's
using it. So it has to have no clones (if clonable) and nobody can
inherit from it (if inheritable). In either case, it must also have
been destructed before it can go away from lack of references.

Here's an example:

Say A inherits from B, and B inherits from C...

<pre>
  C &lt;- B &lt;- A
</pre>

If I destruct C and recompile it, then B is out of date with C.
It's using a previous issue of C. C now has two issues, the old and
the new. If I destruct B and recompile it then the old B still
inherits the old C. But the new B inherits the new C. So:

<pre>
  oldC &lt;- OldB &lt;- A

  newC &lt;- newB
</pre>

If I then recompile A, that means nobody uses the old B or old C
any more. Since I destructed them <i>and</i> nobody's using them,
they'll finally go away. I would then only have one issue of B and
one issue of C.

To spell that out further, let's arbitrarily assign some
instance numbers to the issues.

Say the old A is issue #1, old B is #2, old C is #3. So,

<pre>
  C(#3) &lt;- B(#2) &lt;- A(#1)
</pre>

Now you recompile C (#3). So we have

<pre>
  C(#3) &lt;- B(#2) &lt;- A(#1)

  C(#4)
</pre>

That extra issue of C is just sitting off by itself. Nobody
inherits from it. Then you destruct and recompile B:

<pre>
  C(#3) &lt;- B(#2) &lt;- A(#1)

  C(#4) &lt;- B(#5)
</pre>

When B is recompiled, it looks up C to inherit from it. Issue #4
is the current non-destroyed one, so it finds that instead of #3.
Then, if you recompile A in-place (for instance, if A is clonable
so you don't *want* to destroy it):

<pre>
  C(#4) &lt;- B(#5) &lt;- A(#1)
</pre>

This assumes there's no other objects that reference the old B
or C. If there aren't, then recompiling A (so it looks up B again)
gets rid of the last reference to the old B(#2), which is
destroyed. That removes the last ref to old C(#3), which is also
destroyed. A is now linked to the new ones. If there <i>are</i>
other objects that reference the old B(#2) or C(#3), then A will
still be recompiled as above, but the old B and C will stick around
longer, destructed but active.

Note that since A is recompiled in-place (instead of destructed
and compiled), its issue number stays the same. You can find out
the issue number in the Kernel MUDLib using either:

<pre>
  status(obj)[O_INDEX];

or

  status(path)[O_INDEX];
</pre>

In the first version, status() takes an object pointer which is
for a clonable. The Kernel MUDLib will never give you an object
pointer to an inheritable, so the second version takes the path
string for the inheritable. I don't think the second version can
look up destructed objects, only current non-destructed ones. If
you write an object manager, you have to take that into account and
keep track.

<h3>The Recompile Function</h3>

One thing that may confuse you is the recompile() function in
the driver. Here's an explanation by Erwin Harte of when it would
be called:

<pre>
    * object A inherits objects B and C.</li>

    * both B and C inherit D.</li>

  Now destruct object D and B, and compile both of them again.
  Now you have two different versions of D, one used by B, the
  other used by C.

  It is my understanding that if you would now either 'destruct
  + compile' or 'recompile' A, recompile_object() will be called
  with object C, because that one is still inheriting an already
  destructed object (the original version of D).

  If you do not destruct it, you'll run into an error about
  inheriting different versions of the same object. If you -do-
  destruct it, the inherit_program() will be called in the driver
  object for C that can then use the newer version of D, after
  which A can again inherit it.

  So the bottomline seems to be that recompile_object() is
  called when you're trying to inherit an object that in turn
  depends on an already destructed object.
</pre>

<hr/>

<pre>
Date: Thu, 15 Feb 2001 17:26:32 +0100 (CET)
From: "Felix A. Croes"
Subject: Re: [DGD]kernel cloning and inheriting

Stephen Schmidt wrote:

>[...]
> Let me see if I have this straight; I'm pretty sure at
> some level I don't. Consider three objects, A, Ac, and B.
> Ac is a clone of A; B inherits A.

In the kernel library, inheritables are not clonable, but I
assume that you are talking about the general case.


> 1. You can recompile B because nothing inherits it.
> 2. You cannot recompile Ac because it is a clone; to recompile
>    Ac is basically to recompile A.

Yes.  Recompiling Ac is impossible, but if you could recompile A,
all clones would be upgraded as well.


> 3. You cannot recompile A because B inherits it.
> 4. You could recompile A if you first destructed B, but then
>    object B would be lost. In a persistant world, the loss of
>    B during the recompilation of A could be problematic.
> 5. You could recompile both B and A if Ac did not exist. (?)
>    The root problem is that if you try to recompile A in
>    the presence of Ac, then Ac is forced to change along
>    with B, and Ac might not want to do that.

If Ac did not exist, and A was a pure inheritable object, you could
have upgraded A and B with the following sequence:

    destruct A
    recompile B (will automatically compile A also)

As it is, you can still do this, but of course the state of A will
be lost thereby.  If A was a pure inheritable/clonable without its
own state, you can do this without negative effects on A and B.
Ac, however, will continue to use the old program of A, since A
was destructed before it was recompiled.  Only if A had been
recompiled without deing destructed first -- impossible because of
B -- would ac have been upgraded also.

Regards,
Dworkin
</pre>
<hr>
<pre>
Date: Thu, 15 Feb 2001 18:47:04 +0100 (CET)
From: "Felix A. Croes"
Subject: Re: [DGD]kernel cloning and inheriting

Stephen Schmidt wrote:

>[...]
> > If Ac did not exist, and A was a pure inheritable object, you could
> > have upgraded A and B with the following sequence:
> > 
> >     destruct A
> >     recompile B (will automatically compile A also)
>
> If Ac did not exist, and you recompiled B without first
> destructing A, then A would not be recompiled? I'm pretty
> sure that's right.

Normally, it wouldn't be.  However, A may be out of date as well, in
the sense that it inherits yet another object -- let's call it C --
which has been destructed.  In that case, the recompilation of B
could in itself trigger the destruction of A from the recompile()
function in the driver object; the immediately following recompilation
of B would then also trigger the recompilation of A.


>[...]
> Just out of pure curiosity, why is it not possible to
> have changes in A reflected in both Ac and B? Is it
> because A doesn't keep track of a list of all objects
> that inherit it, so it doesn't know that when it updates
> its own code, it has to update B also? Or is it that when
> A is recompiled, you want B to keep the old behavior? Or
> is there something deeper going on?

Suppose that A is inherited by a further N objects.  If we want
to recompile A, and the changes must be automatically reflected
in those other, inheriting objects, than all of those objects
will have to be automatically upgraded.  I could have made it
that way, and in fact I originally intended to do so.  However,
if N becomes large, recompiling A is going to take a <very> long
time -- minutes in some existing DGD mudlibs.  I wanted to be
able to upgrade A without such a huge delay, so I decided to
divide the upgrade process into many smaller steps, each of which
could be done from LPC, perhaps with a series of callouts.  Since
this was completely new functionality and there was no backward
compatibility to take into account, I decided on the current
limitations.

Regards,
Dworkin
</pre>
