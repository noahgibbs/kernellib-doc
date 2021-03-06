---
title: Security and the Kernellib
layout: default
---

## Why does it apply to <i>me</i>?

You may be thinking to yourself "I'll make <i>my</i> lib secure,
the Kernel MUDLib will be secure, everything will be just fine."
Worse yet, you <i>could</i> be thinking, "the Kernel MUDLib's
supposed to be really secure, so I just won't have to worry about
it." Neither of these is quite true, for reasons I'll go into
later.

Or you <i>could</i> be thinking, "I don't care if it's secure.
I'm never going to have any other wizards working on the MUD that I
don't trust." That's a more valid response, and in fact security
only saves you from malicious users, malicious wizards, and your
own mistakes. If you're fine being left wide open to crashes and
data loss of the last two categories then securing your MUDLib
<i>does</i> get easier.

But not trivial. It's important that you know what even your
users can do to you if you miscode commands. For that reason, you
need to understand how the Kernel MUDLib implements its security
model, and who it lets get away with what. Basically the Kernel
MUDLib only actually prevents a small number of things, it just
restricts the rest to trusted code. That's fine, but then you
<i>need</i> to know what code is trusted and what isn't.

## What code is trusted?

The one-sentence answer is: "code in /usr/System is
trusted."

There's a lot more to it than that. The Kernel MUDLib's own code
is in /kernel, and it's trusted, and you should never, ever modify
it. The Kernel MUDLib will do its best to keep you from doing this
at runtime from a MUDLib written on top of it, but only you can
make sure you don't do so when the MUD's not running. You <i>do</i>
run the machine after all.

Also, trust is delegated in some other ways. There are
directories that are marked as globally readable, so anybody has
read access to them. That also means anybody can clone objects
whose code is in these directories, and several other things that
read access implies. Anybody has full read-write access to their
own directory -- for instance, a wizard named bob will always have
full access to /usr/bob, regardless. There are lots more little
rules like this that we'll hit on a case-by-case basis later.

You could write a function in /usr/System that is callable from
outside /usr/System. If it calls anything that requires trust and
then exports the results, you've just leaked some trust out into
the system at large.

That may be fine. You may have decided that anybody is allowed
to grab the list of clones of /usr/System/obj/room since those
clones do their own security-checking. In that case, you'd be doing
the right thing. You have to leak <i>some</i> trust out into the
system or nobody can do anything at all.

But it's important to understand what people can and can't do
with that information. For instance, in the example above, your
decision means that the code of /usr/System/obj/room.c must be
<i>absolutely bulletproof</i> if it's to be secure from misuse by
your wizards. For instance, say you kept around a function that
would remove an exit from a room but <i>didn't check who called
it</i>. If anybody can get the list of clones, then anybody can
remove any exit from any room, thus isolating your MUD's rooms from
each other. Instead, you'd want to make sure that only a wizard
with full access to the room (such as the one who owns the zone the
room is in) may remove an exit from it.

As an even more severe example, your object in /usr/System might
just keep an array of room clones, and return it to caller. So when
you return the room array an unscrupulous caller could remove
objects from the array. Now whenever somebody else calls the
function to get the array of objects, they won't see the objects
that were removed! Those clones have been entirely gotten rid
of.

Security is a big field, and I'm not going to be able to tell
you everything about it here. Even just DGD and LPC security
specifics is a pretty big field! But you'll need to know what
privileges your code has in order to avoid giving them away to
people that call your functions. So the most important thing these
pages can do is to describe what privileges and what restrictions
the Kernel MUDLib gives so you know what's what.

## General Restrictions

First off, I'll quote Bruce Schneier the security expert. He
says that "security is not something you can bolt onto an existing
product." He's right. The example with the array above should tell
you that you need to think about these things in advance. You may
just be able to copy the array every time, which is fine, but if
you already have code that depends on being able to modify it then
you have a massive headache with no good solutions. Think about
security <i>early</i>!

Now let's get down to things the Kernel MUDLib does differently
than base DGD. DGD itself does almost nothing to enforce access
control, though it's quite secure. By "quite secure" I mean it
doesn't use pointers, so if you don't give somebody access to
something they can't get at it -- they can't go "memory dumpster
diving" easily just because they share your same process. You
pretty much <i>can't</i> crash the machine although if you do
something sufficiently bad DGD won't know what to do and will stop
you.

The Kernel MUDLib adds another layer of robustness onto this. It
realizes that there is good, trusted code and bad, untrusted code
in the world, and that you're likely to have some of both in your
process. So it keeps track of users, and what those users can
access. It keeps track of programs (individual .c files) and
decides what <i>they</i> can access.

For reasons explained <a href="Inheritance.md">elsewhere</a>,
the Kernel MUDLib also adds the <b>library</b> or
<b>inheritable</b> abstraction. It decides that libraries aren't
regular objects like <b>clonables</b>. Instead, the Kernel MUDLib
will never pass you a "real" library object, only its path. That
means you can't keep storage space in one since you can never
create a clone or LWO of one, nor call it with <b>call_other</b>.
All you can really do with a library is to inherit it from your
other code, and in practice that's usually plenty. For situations
where it is not, it's pretty straightforward to simply create a
dummy clonable that does nothing but inherit the library.
<hr>

## Specific Operations

Note: all specifics given here are current as of DGD 1.2.35.
While the Kernel MUDLib tends to change very little, these
operations are, in fact, subject to change at any time by
Dworkin.

Also, this page details only exported APIs -- static and private
functions of the Driver object, for instance, aren't listed even
though DGD or the Driver itself may call them.

* [Auto and Driver Operations](Auto_API.md)
* [Kernel MUDLib Security Case Studies](CaseStudies.md)

<pre>
Date: Mon, 7 May 2001 21:02:59 +0200 (CEST)
From: "Felix A. Croes"
Subject: Re: [DGD]Kernel lib file permissions

Stephen Schmidt wrote:

> I have a question, and I know that the correct answer is
> UTSL, and I will do that eventually, but in the meantime
> I'm hoping someone can give me a general and succinct
> answer: Under the kernel lib, how are file-reading and
> file-writing permissions determined? What objects are
> allowed to read and write files in what directories?
> I know the implementation is done by overriding the
> kfuns in the auto object - the question is what the
> rules are.

It's very simple.  Basically, the file access permissions are much like
in 2.4.5.  By default, wizards only have write access in their own
directory, and read access everywhere outside /usr except /kernel/data.
They may have additional access granted to them.  A wizard's objects
only have that wizard's default access, irrespective of granted permissions.
There are some special cases, such as global read access in /usr/anywiz/open,
but that is the general picture.

File permissions also have an effect on compilation and cloning.  You
can only compile an object if you have write access to the file.  You
can only clone (or inherit) an object if you have read access to it.

Regards,
Dworkin
</pre>
<hr>
<pre>
Date: Thu Aug  9 06:38:00 2001
Subject: [DGD] Access

Pete wrote:

> On 9 Aug 2001, at 13:12, Felix A. Croes wrote:
>[...]
> > That's default object access -- you can only grant access to users.
> > User-level access is handled in /kernel/lib/wiztool.c.
>
> That is what i am speaking about, how should it work? I can grant 
> access for <user> to <dir> but objects from /usr/<user> does not 
> have rights to <dir>, so what is it good for? I have put debug 
> outputs to auto object functions, and it calls access with 
> arguments like:
> user = /usr/World/sys/commandd
> dir = /usr/System/cmd/go.c
> and it does not work even though user World has write access to 
> /usr/System

In the kernel library, objects don't have access outside their own
/usr/Foo directory, even though user Foo may have that access.  This
is intended to prevent security leaks such as the above; if objects
in /usr/World have write access in /usr/System, then effectively
objects in /usr/World can do anything at all.

Regards,
Dworkin
</pre>
<hr>
<pre>
Date: Thu Aug  9 07:53:01 2001
Subject: [DGD] access

Ok, after more trying of it i can only say again its useless...
Sorry people but this is big disappointment for me, since it makes 
admin only admin of wiztool, not admin of whole system. If i would 
have such root on my linux i would throw it away long time ago. Its 
more restrictive then windows nt security, and even that is already 
big crap. I read that kernel library should be good for anything with 
no need for modifications, but this makes it not usable at all i think. 
If I cant grant rights for user objects in runtime then i cant get users 
to cooperate since they cant work with others code in runtime. And 
when i count that any bigger coding simply must be done on local 
copy of mud (who should use mud editor for coding?!) and file 
upload is done by extrenal ftp then the security as it is is simply 
not needed at all.
That means doing rewrite of kernel code, exactly what i did not 
want to do :(
</pre>
<hr>
<pre>
Date: Sun Aug 12 06:45:01 2001
Subject: [DGD] Access

Stephen Schmidt wrote:

> On Thu, 9 Aug 2001, Mikael Lind wrote:
> > After some initial problems I had, Dworkin emphasised that
> > compilation and calling are to be kept separate.
>
> Can someone give some intuition as to why this is desirable?
> I understand it, but don't understand the need for it. I have
> not done much work with the kernel, but it seems to me that
> this method requires that, each time an object calls into
> another object, one must check whether the target object is
> compiled, and if not, pass control to a manager object which
> determines whether to compile and proceed, or abort the call,
> based on the identity of the calling object and whatever else
> you want to take into account.

It is easy enough to simulate the old 2.4.5 behaviour on top of
the kernel library, if that is desired.  But I kept it separate
in the kernel library because compiling and calling are very
different actions in terms of security.

If I create a source file somewhere in my directory with an aim to
eventually make it available to others as a callable server object,
is it not odd that anyone can compile it before I think it is ready?
If it is broken, and I want to repair it, should I not be able to take
it offline without others racking up errors in my code?  An object
compiled from a source file in my directory will be owned by me, and
counts towards my own object quota.  Should others really be able
to increase the number of objects I have in the game, just because
this object is intended to be callable by others once compiled?

This is why compiling an object requires write permission to the
source file.  If you want to make an object available as a server,
it is easiest to just make sure that it is compiled in advance.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Erwin Harte)
Date: Fri Feb  8 16:09:00 2002
Subject: [DGD] Inherit/include

On Fri, Feb 08, 2002 at 09:26:38PM +0000, Shevek wrote:
> I'm noticing some odd behaviour from this the kernel lib (DGD 1.2p1).
> Basically you can't inherit/include from any directory that doesn't have 
> global read access.
> 
> Personally I'd think that if the compiling user has read access to the 
> directory then surely they should be able to inherit/include from it. 
> Doesn't seem to work like that though.
> 
> Is it like this for reasons I'm just not understanding? Because it puts a 
> real crimp on passing code between non global read directories.

The only access that counts at that point is the access that the file
has, it doesn't matter that you (the coder) have a bit more access
than that, otherwise you could have the following situation:

- You (bar) grant <foo> write-access in your home-directory to work on
  something.
- You, one of the main game-admins, also have full access to /.
- <foo> now writes a bit of code in your home-directory and, using
  that, has full access to /.

Sweet, isn't it?

Use your ~/open/ directory if you want to share some feature/interface
information, I'd suggest creating ~/open/lib/ and ~/open/include/ for
this purpose and for any code you want to share you can put an almost
empty inheriting bit in ~/open/lib/fnurt.c which can be used by
others.

Consider it the 'black box' approach. :-)  If you want to share all of
the code then put all of it in your ~/open/ directory.

Hope that helps,

Erwin.
-- 
Erwin Harte
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Shevek)
Date: Fri Feb  8 18:11:01 2002
Subject: [DGD] Inherit/include

>The only access that counts at that point is the access that the file
>has, it doesn't matter that you (the coder) have a bit more access
>than that, otherwise you could have the following situation:

 From what I see using status in the kernel lib the owner of a file gets 
set to wherever it is compiled from it just doesn't seem to change a great 
deal of anything. There isn't anything insecure about letting a file take 
the access of the user who compiles it, so long as the user doesn't go 
around compiling code they haven't checked.

Things are stranger than it just being the file's access that's odd though.

Eg I copied a file called test into ~/System, which contained a single 
command to write a file into the ~/System directory. On compilation with 
any user that has access to System it quite happily writes a file into the 
~/System directory. Tried this again with a user that has write access to 
~/System but not write access to ~/Private. Upon compilation in ~/System 
the test program can merrily write its test file into ~/Private.
To me that sounds exactly like the behaviour you describe below, with the 
test program taking System's root access.

But it gets more complex. When you give a user (I'll call him Test) write 
access to a directory (I'll call it ~/Public). Test can use the editor to 
make files in ~/Public, but can't compile code in their home directory that 
writes files to ~/Public.
Now say Test is given read access to a directory (I'll call it ~/Private) 
they can read anything they like, but any code they write in ~/Test can't 
read from the directory (Although it can if ~/Private is global read).
Ie the file access problem you pointed out.

Effectively this makes any code, written in a user directory that isn't 
~/System, trapped in its own directory, only able to read from global read 
directories, with any code written in ~/System having full root access 
(Despite includes/inherits from anything other than the kernel).

I can see why trapping code inside the directories is secure, I just think 
that if code can access that which the owner (Ie compiler of the code) can 
access then it makes things a whole bunch easier to use. Otherwise anything 
that has to read/write from outside its user directory (From anything that 
isn't global read) has to go into System, or use a daemon in System to give 
it file access outside its user directory.

>- You (bar) grant <foo> write-access in your home-directory to work on
>   something.
>- You, one of the main game-admins, also have full access to /.
>- <foo> now writes a bit of code in your home-directory and, using
>   that, has full access to /.
>
>Sweet, isn't it?

Define sweet :>

I get your point, although that wasn't the behaviour I was suggesting.

If I have code in System, and get it to be compiled with System as owner 
then it has full / access anyway. Just can't inherit/include anything from 
anywhere but ~/include or a global read dir which still seems bizarre to 
me. So I can have a piece of code which is able to read/write/delete any 
file it likes (Tested this), but cannot inherit/include the file. Does that 
not sound even a little odd to you?

>Use your ~/open/ directory if you want to share some feature/interface
>information, I'd suggest creating ~/open/lib/ and ~/open/include/ for
>this purpose and for any code you want to share you can put an almost
>empty inheriting bit in ~/open/lib/fnurt.c which can be used by
>others.
>
>Consider it the 'black box' approach. :-)  If you want to share all of
>the code then put all of it in your ~/open/ directory.
>
>Hope that helps,
>
>Erwin.

Do this already with stuff like a room inheritable etc.
Thing is all I want to do is inherit/include from a private directory that 
doesn't give any special write access :>

Cheers,
         Shevek
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Sat Feb  9 07:49:02 2002
Subject: [DGD] Inherit/include

Shevek wrote:

> >The only access that counts at that point is the access that the file
> >has, it doesn't matter that you (the coder) have a bit more access
> >than that, otherwise you could have the following situation:
>
>  From what I see using status in the kernel lib the owner of a file gets 
> set to wherever it is compiled from it just doesn't seem to change a great 
> deal of anything. There isn't anything insecure about letting a file take 
> the access of the user who compiles it, so long as the user doesn't go 
> around compiling code they haven't checked.

Foo examines a bit of code and finds it to be secure; Foo has gotten
into the habit of checking every bit of his own code just before he
compiles it, because there are others who have write access to his code.
Foo is very security-conscious, and to make extra sure, he checks it
three more times.  However, just before Foo actually compiles the code,
Bar replaces it with a version of his own.  Bingo, a security leak.


> Things are stranger than it just being the file's access that's odd though.
>
> Eg I copied a file called test into ~/System, which contained a single 
> command to write a file into the ~/System directory. On compilation with 
> any user that has access to System it quite happily writes a file into the 
> ~/System directory. Tried this again with a user that has write access to 
> ~/System but not write access to ~/Private. Upon compilation in ~/System 
> the test program can merrily write its test file into ~/Private.

Never grant anyone write access to ~System.  Always grant then full access
to / instead.  You're right, it amounts to the same thing, because any code
in ~System has full rights anyhow.


> To me that sounds exactly like the behaviour you describe below, with the 
> test program taking System's root access.
>
> But it gets more complex. When you give a user (I'll call him Test) write 
> access to a directory (I'll call it ~/Public). Test can use the editor to 
> make files in ~/Public, but can't compile code in their home directory that 
> writes files to ~/Public.

Test however can write and compile files in ~/Public that can read and
write anything else in your directory, because objects in your directory
can access anything else in your directory.  Bingo.  This is probably not
what you intended, but if it's any consolation, Test's access to your files
does not extend to the files outside of your directory that you, as a
user, have special access to.  Though you inadvertedly gave away access
to all of your files, you didn't endanger anyone else by doing so.

There is not much point in giving someone write access to a subdirectory,
only.


> Now say Test is given read access to a directory (I'll call it ~/Private) 
> they can read anything they like, but any code they write in ~/Test can't 
> read from the directory (Although it can if ~/Private is global read).
> Ie the file access problem you pointed out.

Hardly the same problem that Erwin mentioned.


> Effectively this makes any code, written in a user directory that isn't 
> ~/System, trapped in its own directory, only able to read from global read 
> directories, with any code written in ~/System having full root access 

Exactly!  This is intentional.


> (Despite includes/inherits from anything other than the kernel).

But of course, nothing in ~System should ever depend on any code outside
~System, since such code could introduce a security leak.


> I can see why trapping code inside the directories is secure, I just think 
> that if code can access that which the owner (Ie compiler of the code) can 
> access then it makes things a whole bunch easier to use. Otherwise anything 
> that has to read/write from outside its user directory (From anything that 
> isn't global read) has to go into System, or use a daemon in System to give 
> it file access outside its user directory.

Generally, don't write code that requires access outside its own directory.
The only object that has a good reason to do so is the wiztool, which is
why an object in ~System can inherit /kernel/lib/wiztool (which has already
masked all the relevant functions so that only user-level access, and not
System-level access, is possible).


>[...]
> If I have code in System, and get it to be compiled with System as owner 
> then it has full / access anyway. Just can't inherit/include anything from 
> anywhere but ~/include or a global read dir which still seems bizarre to 
> me. So I can have a piece of code which is able to read/write/delete any 
> file it likes (Tested this), but cannot inherit/include the file. Does that 
> not sound even a little odd to you?

No object in ~System should depend on code inherited or included from
elsewhere (other than /kernel), since such code could then do whatever
it wants.  Bingo.  Any code that is to have System rights should be in
~System.  If I wanted to remove the oddity, I would forbid the
dependency altogether, rather than allow it.


>[...]
> Thing is all I want to do is inherit/include from a private directory that 
> doesn't give any special write access :>

Sorry, but you cannot use kernel lib file access in this manner.  If you
want to restrict access to certain functionality, you have to do so at
runtime.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Noah Lee Gibbs)
Date: Sat Feb  9 18:10:01 2002
Subject: [DGD] Inherit/include

On Sat, 9 Feb 2002, Felix A. Croes wrote:
> Could it have been the other way?  First you created an explicit System
> user, and then you gave it extra access.  It would be hard to fathom why
> you wanted to use the wiztool in this situation, unless you had a user
> already.

  Any call to compile_object in the wiztool goes through the wiztool
compile_object function, regardless of whether it reflects user
input.  In this case, it does not.

> I do not think it is a good idea to make a System user who can login.

  There isn't a System user who can log in.  Indeed, that is explicitly
forbidden.  However, in the create() function of the wiztool it compiles
the objects it may later wish to clone.  In my case, they are things like
/usr/common/obj/user_state.c.  When I include a call to compile_object on
that object in the creator function, the call to access() that occurs in
/kernel/lib/wiztool.c passes owner (System), path
(/usr/common/obj/user_state.c) and access (WRITE_ACCESS).  Therefore,
System winds up needing write access to that file to compile it in its
create function.

-- 
See my page of DGD documentation at
"http://www.angelbob.com/projects/DGD_Page.html"
If you post to the DGD list, you may see yourself there!
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Mon Nov 17 17:55:01 2003
Subject: [DGD] Kernel Lib security and APIs

Noah Gibbs wrote:

>[...]
>   I like this in concept.  It's working for me. 
> However, I have a couple of questions about how to
> make it work properly with Kernel Library security.
>
>   I'd like to keep (for instance) room files somewhere
> under /usr/game, and then pass the filename to a
> routine in /usr/common.  The routine in /usr/common
> would open and parse the file and put the rooms in
> their appropriate places.  It would read script names
> out of the file and call those scripts.  The scripts
> exist in one or more other files in /usr/game.
>
>   There's a problem with this -- there's no way that I
> can see to get an object under /usr/common to read a
> file under /usr/game without using a globally-readable
> directory.  I tried giving the "common" user full read
> access to /usr/game, but it made no difference.

Directories under /usr serve three different purposes.

First, security: user directories are created there.  Each user
normally only has access to his own files.  Global read access can
be added for a directory, which allows all users and their objects
read access to files in that directory.

User-specific access applies <only> to the user, not to his objects.
If you really want objects to access files in a different directory
under /usr, make a server object in that directory which hands out
files on request.  For your example above, /usr/game objects should
really pass the file contents to /usr/common objects, not the file
names.

File security was designed with proper restrictions for untrusted guest
programmers (wizards) in mind, and it may get in the way sometimes
in muds which are organized differently.

Second, resource control and measurement: this one is very important.
Each "user" under /usr (whether representing an actual user or not)
has his own resource limits.  With guest programmers, you probably
want these to be actual limits so nobody can clone arbitrary amounts
of objects, create callout explosions, hand out huge amounts of money
to players, and so on.  Without them, you'll probably still want to
use them for measurement, to see how many objects/callouts/ticks/etc
are used in a specific section of your mudlib.  It is recommended to
structure your mudlib into different directories under /usr with this
in mind.

Note that file access is based on the object creator (based directly
on the file name), whereas resource management is based on the object's
owner.  For master objects, the object owner is always the same as the
object creator.  For clones, the owner is the same as the owner of the
object that cloned it.  So if an object in /usr/foo clones an object
from /usr/bar, the owner of that object is foo.

Third and last, concurrency: the kernel library keeps track of a lot
of things for you.  Unfortunately this can have an adverse effect in
DGD/MP; with a single server object keeping track of things, two LPC
threads using those things cannot run concurrently, because they
effectively cause modifications in that same server object.

The kernel library works around this in two different ways.  Decaying
resources (such as tick measurement) are not affected.  Non-decaying
resources (such as the number of objects) can be modified concurrently
if and only if this is done from objects with different owners.  So
here lies another reason to have different subsystems in different
directories under /usr.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Fri Dec  5 07:41:01 2003
Subject: [DGD] Cross-directory inheritance and read access

Noah Gibbs wrote:

>   In the Kernel Library, you need to have read access
> to a file in order to inherit it.  Special access
> (giving a user specific access to a specific
> directory) doesn't work for files in those users'
> directories.  So in order to inherit from a library,
> that library has to be globally readable.
>
>   That seems wrong to me.  The only method to prevent
> somebody inheriting a globally readable library is the
> forbid_inherit mechanism in the ObjectD.  You could do
> that, but it's a fair amount of work, and it's
> circumventing the existing permissions system --
> you've already made sure that they can read the file,
> because if they can't then they can't inherit it.  I
> suppose Phantasmal could demand write-access to a file
> in order to inherit it, but that would be *really*
> insecure.
>
>   I could just skip the inheritance and do all work by
> replacing the child object with a hook object, and
> passing the calls through to it.  That seems like a
> very awkward interface, though.  Is there some way to
> reasonably access-control inheritance without making
> directories like /usr/System/lib globally readable, or
> moving the libraries to /usr/System/open/lib?

What exactly do you want: have files readable without making them
inheritable, or have files inheritable without making them readable?

In the first case, you should use forbid_inherit to define your own
security model.  This doesn't have to be complex.  For example, you
could prevent inheriting anything that has "/private/" as a pathname
component unless the inheritor has the same creator.

In the second case, you can separate file access and inheritance by
making ~/open/lib/foo.c inherit from ~/lib/foo.c.  Note that it is
a bad idea to do this for a second-level auto object, which is
inherited by everything, because you'd needlessly pollute the
inheritance tables that DGD uses internally.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Wed Jan 28 07:06:01 2004
Subject: [DGD] Alternatives to the Kernel model of security...

Noah Gibbs wrote:

>   The Kernel Library, while it's a very powerful and
> useful bit of software, is undeniably hard to use in
> certain cases.  I don't mean it's technically
> incapable.  I mean that its security model is
> unfamiliar to essentially everyone, and that common
> forms of security are difficult to map onto its
> interace.

I'll explain a little about how the kernel library's security model
came to be.

Before I started working on DGD, I spent a lot of time trying to
circumvent the security of various mudlibs.  When I started, all LPmuds
used a variety of 2.4.5 mudlib, and eventually I developed a routine
to go from newbie player to archwizard in about 15 minutes (including
removing evidence from the logfiles). :)

The basic idea of LPmud 2.4.5's security was simple: wizards can only
edit files in their own directory, in /open and in directories where
they have explicitly been granted access, archwizards can edit files
everywhere except in /room and /obj.  The implementation was full of
holes.

A new security model was developed for the CD mudlib.  The main change
was that security became object-based, instead of player-based.  This
introduced the wiztool problem: when wizard A cloned a wiztool made
by wizard B, that wiztool could not access files in the directory of
wizard A.  To solve this, there had to be a way for wizard A to grant
his cloned wiztool permission to access files in his own directory.

This was managed with the uid (user ID) security system, partially
implemented in the server and partially in the mudlib.  The uid
system was a disaster, for various reasons:

 - the design was based on the Unix suid (set user ID) system, which
   is notoriously bad
 - the implementation was so complex that problably only Genesis
   archwizard Commander understood it, but a number of others were
   nevertheless making changes to it, and the system became less and
   less secure with each new mudlib release
 - overall security was affected by a huge number of source files,
   all of which had to be in tune with eachother
 - The 3.1.2 LPmud server, with builtin uid support, was used as the
   basis for the MudOS server, for which new mudlibs were developed
   which tried to make a different use of the existing uid support
   in the server, and which all failed miserably.

I have broken uid security so many times that eventually, the
archwizards of Genesis simply stopped fixing the bugs that I
reported to them.

Then came what came to be known as the "stack-based" security system,
which was an enormous improvement.  This is the system that is still
used for the Lima mudlib today.  Though I managed to break it on one
occasion, I never found a fundamental flaw in it.  I do have some
reservations.

Stack-based security does not depend on the current player (as in
2.4.5), or the permissions of the current object (as in CDlib), but
on the intersection of the permissions of all objects in the call
chain.  Each object has the option of "resetting" security before
performing a sensitive operation, with the effect that this operation
will be performed with only the permissions of the current object
taken into account.

The problem of stack-based security is that there are many objects
which you want to be transparent with regard to security, i.e. you
want them to be able to appear in the call chain without affecting
the combined permissions.  Unfortunately, the only way to accomplish
this is to give the "transparent" object maximum permissions.  In the
Lima mudlib today, there is a large set of "transparent" objects, all
of which are innocuous in function, and dangerous in potential.
Finding a security problem now involves finding a badly-made
"transparent" object.

For the kernel library, I wanted a security model with the following
additional properties:

 - the set of objects that require maximum permissions would be very
   small
 - a breach of one programmer's security does not affect the security
   of the remainder of the system, regardless of that programmer's
   permissions

In the kernel library, only the kernel objects that define the
security model itself have maximum permissions.  System objects have
almost maximum permissions; the one thing they cannot do is change the
security model.  The set of System objects can be small (I have less
than 20 in my own mudlib).

To meet the second requirement, I separated the permissions granted to
a programmer, and to that programmer's objects.  After making this
separation, I decided that stack-based security would now be needlessly
restrictive.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Fri Feb 27 05:49:01 2004
Subject: [DGD] Kernel Lib, creator & owner

Michael McKiel wrote:

> Well I tested some more, logged in as admin.
> copied, blah.c to /usr/zamadhi/obj/blah.c
> compiled it, cloned it.
> The master object winds up with zamadhi as creator & owner.
> And the clone winds up with "admin" as owner, and zamadhi as query_creator()
>
> Does that present any security consequences then? for say the purposes of the
> Master object being able to destruct the clone, or to update the clone? when
> "admin" becomes the owner?

The object is owned by 'admin' and can only be destructed by that owner
or by someone with administrator access.  It also counts towards admin's
object quota, if any.  File access is still governed by creator.  Other
resources are all used in admin's name.

Regards,
Dworkin
</pre>
