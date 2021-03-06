---
title: Kernel Library Email Archive
layout: default
---

Everything in here was culled from the mailing list. These
should all be interesting Kernel-Library-related messages. There's
a lot of them.

<pre>
From felix Sun Aug  2 16:44:04 1998
To: dgd@list.imaginary.com
Subject: Re: [DGD] Re: DGD LPC (?)

Mikael Lind wrote:

>[...]
> The problem with your solution is that get_dir()'s wildcard matching is
> ignored...
>
> get_dir("/a/*") returns the files "b", "c" and "*" but your
> file_size("/a/*") returns -1. (Should return the size of "/a/*".) Probably
> not a problem if your files have names clean from wildcards.
>
> get_dir("/a/*") returns the file "b" and your file_size("/a/*") returns the
> size of "/a/b". (Should return -1.) Might be a problem.

The continuing confusion about get_dir() is probably an argument in
favour of of putting a file_info() function in the kernel lib's auto
object, returning get_dir() info for a single file, with wildcards
properly avoided.


Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Sat Dec 15 17:47:01 2001
Subject: [DGD] kernel lib wiztool->expand()

Shevek wrote:

> Ok, now the problem.
> What does expand() return, and am I even using it right? I know for a fact 
> it expands a filename into a full path, but I see that it can also do some 
> access checking.

mixed *expand(string files, int exist, int full)

files:      examples: "/foo/bar", "/foo/*", "/foo/bar /foo/gnu *.c"
exist:  -1: files need not exist
     0: all files have to exist, except for the last one
     1: all files have to exist
full:       TRUE for full pathnames in the result

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Shevek)
Date: Tue Dec 18 13:32:00 2001
Subject: [DGD] More info on expand()

Had another look at expand() and managed to clean my code up quite 
considerably.
This may come in handy for someone, somewhere, so I thought I'd leave it 
here in case anyone else has a similar problem.

In:
Path to filename from current directory, or just filename if it's in 
current directory.
Eg System/initd.c if in ~/. initd.c if in ~/System.

Out:
Nil if a directory/file doesn't exist/access denied/wildcards used.
Full path to file if not a directory/exists/access allowed/single filename.

/*
  * NAME:   single_file_path()
  * DATE:   18/12/01, Shevek.
  * DESCRIPTION:    Returns the full path to an existing file, nil if there is 
an error
  */
static string single_file_path(string str){
        mixed *info;
        info = expand(str, 1, TRUE);
        if (info[4] == 1) {
            if (sizeof(info[0]) != 1) {
                return nil;
            }
            if (info[1][0]==-2){
                message(str + ": Is a directory.\n");
                return nil;
            }
        }   
        else {
            message("Error: Use single filenames only.\n");
            return nil;
        }
        str = info[0][0];
    return str;
}

Requires:
message() : return message to user.
expand() from /kernel/lib/wiztool plus whatever expand needs.

Hope that helps someone out.

Cheers,
    Shevek
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Shevek)
Date: Sat Feb 16 22:23:01 2002
Subject: [DGD] Kernel Mudlib

At 09:08 16/02/02 -0800, you wrote:
>On Fri, 15 February 2002, Erwin Harte wrote:
> >
> > Return MODE_DISCONNECT or whatever it's called as the very first thing
> > in that special user-object, then the connection will be closed almost
> > immediately.
> >
>
>OK thanks.  I think I might be doing that as soon as login() is called on 
>the user object.  I was just wondering if there was some way of disabling 
>the telnet port without a special user-object.

Set the timeout in telnetd or binaryd (Or whatever you pass via initd.c) to 
-1, effectively this disconnects as soon as they connect. No need to pass 
any object back as it's never used.
It's good to be nice about it and leave query_banner giving some 
explanation as to why they can't logon on the telnet/binary port.

Eg
object select(string str) {
         /* No need to return anything, but you need the function I think */
}

int query_timeout(object connection){
         /* Disconnect immediately */
         return(-1);
}

string query_banner(object connection) {
         /* Needed, it gets called before query_timeout */
         string str;
         str="\nTelnet connections are not permitted.";
         str+="\nUse port 6048 \n";
         return(str);
}

Shevek
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Fri Mar 15 20:33:00 2002
Subject: [DGD] Kernel question

Jay Shaffstall wrote:

>[...]
> My first question is that DGD's send_message kfun is supposed to send a 
> message to the current user.  The kernel mudlib connection object provides 
> a message () method that wraps send_message, seemingly to provide for 
> automatic resending of anything that doesn't make it out on the first 
> send_message.  In what circumstances will an entire string not make it to 
> the user on the first send_message?

An entire string will always make it on the first call, but possibly
not on the second one, because DGD's internal buffer is only one
string large (and strings have a maximum length).


> The connection object also seems to assume that a second call to 
> send_message will always send everything that's still pending (this is done 
> in message_done).  Is that true?

No.  If you'll think about it, you'll see that the send_message call from
message_done() is always the <first> call -- the first one in the current
thread.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Par Winzell)
Date: Tue Mar 19 11:06:00 2002
Subject: [DGD] New to List - Some questions

Jay Shaffstall wrote:
> What really helped me, though, was sitting down and writing a driver 
> object from scratch (using the kernel mudlib's and melville's driver 
> objects as comparisons for different ways of implementing the methods 
> that are required, and borrowing some kernel code for getting the stack 
> trace for runtime errors).  The comparison process really helped me to 
> get a handle on the purpose of each method.
> 
> Don't worry too much about the auto object at first.  It's the DGD 
> equivalent of Java's Object (i.e. the object everything else inherits 
> from), and can be blank until you come up with functionality that must 
> be common to all objects.
> 
> Also, when you're looking at the kernel mudlib, remember that its 
> purpose is to provide a secure environment for multiple coders to work 
> in together.  If that isn't part of your game, then much of what the 
> kernel does won't be needed for you.
> 
> Finally, since I'm a rank newbie at DGD as well, my comments may be 
> entirely based on my own misunderstandings, and are subject to 
> correction by the more experienced folks here.

This is all excellent advice. The 'immerse self by doing' approach is 
easily the best one for DGD. It wouldn't hurt to have a FAQ somewhere 
explaining some of the things you discover and wonder about in the 
process of writing your first auto/driver objects, but this list is a 
pretty good resource for that, too.

The kernel mudlib takes a pretty powerful investment of time to really 
comprehend. It may not be the ideal start for somebody approaching DGD 
for the first time. Understanding the kernel library -does- give you an 
understanding of DGD, of course, but it's rather overkill and one could 
argue that DGD's main strength is its ability to be so powerful while 
avoiding directing the developer in any one given direction. The kernel 
library does direct you somewhat and you may lose some freedom of 
thought if you immerse yourself in the kernel library immediately.

Writing a driver/auto object first yourself may be a better idea, yes.

Zell
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Mon Dec 17 19:32:01 2001
Subject: [DGD] More info on expand()

Wes Connell wrote:

> I'm not sure if Shevek solved his expand() problems, but I am having one of
> my own. Inside expand() the variable 'dir' is initially set by get_dir(). 
> The kfun docs on get_dir() say that it returns an array in the following
> configuration:
>
> ({ ({ file names }), ({ file sizes }), ({ file mod times }) })
>
> However, near the end of expand() the following code is executed:
>
>         all[0] += dir[0];
>         all[1] += dir[1];
>         all[2] += dir[2];
>         all[3] += dir[3];
>         all[4] += sizeof(dir[0]);
>
> This is expecting a 4th value inside 'dir'. Some parts of the expand() code
> do set this 4th value but only in the case of error.

No, the 4th value is copied straight from dir[3].  It's the 5th value that
is set in case of an error, and also by the last line of the code you
quote.


> I know this 4th value is used to determine whether or not the filename is
> currently compiled into an object. The only place I've found this used is 
> in the ls command to insert the asterisk before the filename.
>
> So my quesiton is, how is this 4th value set? I have exported the code to
> my own wiztool but I dont think that should matter much since it is calling
> get_dir() as a kfun.

It's calling get_dir() as a function in the auto object, not as a kfun.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Par Winzell)
Date: Fri Dec 14 20:18:01 2001
Subject: [DGD] kernel lib wiztool->expand()

Shevek writes:

 > What does expand() return, and am I even using it right?

Maybe someone else can answer this. Since you have the source
available, though, I figure you can probably guess just about
as well as anybody else except for Dworkin who pay possible
recall what it's supposed to do. :-)

 > 1) What is Ecru? I know it's a user, but why?

It collects everything for which there is no explicit user.

 > What I'm actually trying to do is something like this:
 > filename=SOME_DIR + "/" + (First char of str) + "/" + str + ".pwd";
 > where str=someone's name for the purposes of seperating files into 
 > directories based on first letter of name. Ie all the people starting with 
 > 'A' into ~/System/data/A.

You would benefit greatly from increased use of capitalization.

I think you want

    filename = DIR + "/" + str[0 .. 0] + "/" + str + ".pwd";


Zell
</pre>
<hr>
<pre>
Date: Fri, 20 Apr 2001 11:46:03 -0400
From: Kris Van Hees
Subject: Re: [DGD]Initializing problem

On Fri, Apr 20, 2001 at 05:38:22PM +0200, Erwin Harte wrote:
> Not sure.  I would probably also include defines in that file that
> could be useful for objects in other /usr directories, at which point
> I'm not sure which is easier/better to work with, this one:
> 
>   # include <System.h>
> 
> Or this one:
> 
>   # include "~System/include/System.h"
> 
> I think the /include/System.h file should give general pointers into
> the /usr/System/ tree, while ~System/include/ would contain include-
> files for specific objects' interfaces.

An alternative might be to add ~System/include to the includes list in the DGD
config file?  Potentially before others if you intend on possibly overriding
anything that gets defined in the kernel lib.

    Kris
</pre>
<hr>
<pre>
Date: Fri, 20 Apr 2001 18:27:40 +0200
From: Erwin Harte
Subject: Re: [DGD]Initializing problem

On Fri, Apr 20, 2001 at 10:56:23AM -0500, Jason Cone wrote:
[...]
> 
> For each of the 2 locations, /kernel/include/System.h and
> /usr/System/include/System.h, the statement,
> 
>   #include <System.h>
> 
> , is still a valid declaration; I verified this by moving your System.h from
> /kernel/include to /usr/System/include and changed nothing else.  Because
> one of the (secondary) include paths in the config file is "~/include", any
> file that is represented by "/usr/<single directory>/include/<file>" can be
> included via "#include <file>".

Try including it from, say, ~admin/foobar.c, and you will find that
the ~/include/ directory is assumed to be ~admin/include/, not
~System/include/.

[...]
> 
> To me, a custom secondary library should be totally and completely
> encapsulated by the /usr/System -- that way, alternative core libraries
> could be developed on top of which any library that was configured to run on
> top of Felix's core library could also run (which may, or may not, contain a
> '/kernel/include' directory).

The /kernel/include/ directory would only be a problem if
/kernel/include were part of the search-path, which it isn't, adding
it is what I would consider an interface-change and all bets are off
at that point. :-)

I understand your point about the encapsulation, if I were to further
develop this particular demo-System-lib I might change things a bit.

Regards,

Erwin.
-- 
Erwin Harte      : `Don't mind Erwin, he gets crabby. :)'
</pre>
<hr>
<pre>
Date: Tue, 3 Apr 2001 23:00:37 +0200 (CEST)
From: "Felix A. Croes"
Subject: Re: [DGD]Using the kernel lib

Stephen Schmidt wrote:

> Question on the kernel lib: What is the division of code between
> /kernel/lib/user.c and /kernel/obj/user.c? I'm hoping that everything
> "important" is in /kernel/lib/user.c, and that one can remove and
> replace /kernel/obj/user.c without causing any problems. Is that,
> indeed, the case? Or are there things in /kernel/obj/user.c which
> must be there for the kernel lib to function?

The kernel lib's user object is <meant> to be replaced, but the
replacement must have certain functions -- create(), login(), logout(),
receive_message(), and optionally receive_datagram() -- to function
properly.  It also must inherit /kernel/lib/user.

Regards,
Dworkin
</pre>
<hr>
<pre>
Date: Tue, 3 Apr 2001 23:34:39 +0200 (CEST)
From: "Felix A. Croes"
Subject: Re: [DGD]Using the kernel lib

Stephen Schmidt wrote:

> On Tue, 3 Apr 2001, Felix A. Croes wrote:
> > The kernel lib's user object is <meant> to be replaced, but the
> > replacement must have certain functions -- create(), login(), logout(),
> > receive_message(), and optionally receive_datagram() -- to function
> > properly.  It also must inherit /kernel/lib/user.
>  
> OK. I presume the same is true of /kernel/obj/telnet.c; you
> replace it with something else that inherits connection.c
> in the lib directory. What functions have to be defined
> in that file?

No, /kernel/obj/telnet is not intended to be replaced, and neither is
/kernel/obj/binary.

Regards,
Dworkin
</pre>
<hr>
<pre>
Date: Thu, 19 Apr 2001 22:37:08 +0200 (CEST)
From: "Felix A. Croes"
Subject: Re: [DGD]Initializing problem

Stephen Schmidt wrote:

> On Thu, 19 Apr 2001, Felix A. Croes wrote:
>[...]
> > First off, making modifications to the kernel library is a bad idea.
>
> But a couple weeks ago I was told:
>
> >>> The kernel lib's user object is <meant> to be replaced

Replacing is not modifying :)  The kernel library allows you to replace
the user object, by creating a telnet connection manager or binary
connection manager.  I have detailed how to do this in a recent posting,
in response to a question of your own I believe.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: Par Winzell
Date: Fri, 20 Apr 2001 10:27:13 -0700
Subject: RE: [DGD]Core library and initd.c

 > Doh!  I failed to add 2 + 2 correctly.  In this case, perhaps it does make
 > sense for there to be a /kernel/include/System.h-like include file that can,
 > if present, inform the driver (/kernel/sys/driver.c) of the locations of all
 > secondary library-specific files (though, at present, the initd.c file is
 > the only that comes to mind).

In my opinion Dworkin's great strength is he masters the full spectrum,
from tinkering in near-optimal assembly code to intuiting architectural
designs, and through this mastery is the slave of neither extreme. This
results in code and design that is a very unusual mix of flexible and
generic on one hand, and finalized and settled on the other.

I think ~System is one of these cases. For the kernel library to fully
stand alone, it is appropriate for it to perform one well-defined call,
placing the onus of integration entirely in the hands of the integrator.

Modifying an include file in /kernel has a very different feel; it has
the effect of 'pointing' the kernel library at your code. Sometimes, for
some designs, that is a useful thing: e.g. DGD itself works that way.

But for the kernel library it it inappropriate. The self-containedness
of the kernel library is the main reason it is interesting to me. When
every piece of code I ever wrote has crashed, I can still log into the
emergency admin port and fix the problem using code that I know has not
been touched through my frenzied development. It's a great boundary to
have -- to be able to rely on: no files go in /kernel. period.

If you really wanted to change System to something else, the place for
it would probably be in /include/config.h, but I advice against that.

Zell
</pre>
<hr>
<pre>
Date: Sat, 12 May 2001 02:39:49 +0200 (CEST)
From: "Felix A. Croes"
Subject: Re: [DGD]Auto object

I wrote:

>[...]
> You've been putting far too much in /usr/System.  Most of this should
> probably be in /lib, /sys, /obj, or in some directory I haven't thought
> of yet.  Since objects not in /kernel or /usr have very restricted
> file and object permissions, some of it should perhaps be in
> /usr/Melville.
>
> From what you've told us so far, I see a need for 3 objects in
> /usr/System: initd.c, sys/objectd.c and lib/auto.c.

All right, I was being too minimalist here.  You also need a telnet
connection manager and a user object in /usr/System.

So, what should be in which directory?  Objects outside of /kernel and
/usr cannot do file operations or create objects, so I would suggest
something like the following:

/sys        contains stateless "daemon" objects
/lib        contains inheritables that are useful to all wizards
/obj        contains clonables that are useful to all wizards
/usr/Melville   contains the first few rooms of the mudlib; rooms must
        be able to create other objects such as monsters, and
        therefore should be inside /usr.

Alternatively, you could directories such as create /melville/sys,
/melville/lib, etc.

You may find a need for other objects in /usr/System, but you should
only put objects in there if such a need exists.  Any object in
/usr/System can write anywhere except in /kernel, can create objects
that are owned by any other user, and can destruct any object that
is not in /kernel.  The more objects you have in /usr/System, the
more objects you have to check for possible security holes.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: Par Winzell
Date: Wed, 23 May 2001 06:38:13 -0700
Subject: Re: [DGD]Kernel mudlib question: What's TLS?

 > TLS = Thread-local-storage.
 > 
 > The idea is to have a storage-place that is unique for each thread but
 > without the use of one global object.  The reason for that is that
 > such a global object would cause multiple threads to 'abort' eachother
 > when they both want to modify data in the same object, while the
 > individual TLS can be modified by the relevant threads without causing
 > such conflicts.

Some examples:

 *) The kfun this_user() returns nil in a callout. At times, one would
    want this value preserved across callouts, since it's usually well-
    defined at the time the callout is started. The Skotos library now
    does this, and it uses a TLS slot to store the preserved value for
    the duration of the thread.

 *) If DGD did not count ticks itself, we could provide a rudimentary
    execution-time limiter by updating a TLS slot whenever we did some
    big operation (or the cheesy version: store time() at the start of
    the thread and occasionally compare it to time() so that threads
    on average freeze the process for no more than 0.5 seconds).

 *) You could designate a TLS slot as 'debug data', perhaps putting a
    mapping value there or maybe an array of mappings or whatnot; and
    as the thread weaves its way through the code of your library,
    debug functions could place interesting data there which gets
    stored or logged or somesuch at the end of the thread, perhaps
    under certain conditions...

Of course, DGD does not yet have multi-processing, but this is one of
the many preliminary architectural features that Dworkin has put there
in preparation. Also, it's just tidy to keep stuff that is relevant in
the context of a thread private to that thread.

Zell
</pre>
<hr>
<pre>
Date: Sat, 26 May 2001 04:10:44 +0200
From: Erwin Harte
Subject: Re: [DGD]Inherit and include

On Sat, May 26, 2001 at 02:48:59AM +0100, mtaylor wrote:
[...]
> Dear list ...
> 
> Another set of questions from me ;)
> 
> We have built our mudlib by looking at melville, 2.4.5 and the kernel with
> the DGD driver so we are learning as we go. Everything seems to be fine and
> working well but I am unsure of a few things.
> 
> Firstly I am having a problem with the close() function. We have a player.c
> which holds all the information for the Players Character (name, stats etc)
> and also the functions that deal with the connection stuff. Is this a bad
> idea? I've noticed that other mudlibs have a user.c and player.c - one holds
> the connection functions and one the functions for that player's character
> in the world. 

Yes.  Reason being that this way it is possible to (a) take over from
another connection and re-establish your relation to the same player-
object with a new user-clone, and (b) in a persistent game you don't
want to have to destroy your player-object when you need to forcibly
close the connection.

> Now if I have a close function in the player.c I have two big problems. One
> is that if I call destruct_object(this_object()) then it says Too many
> arguments for function close.

Most likely you've defined the function as 'void close() { ... }'
while it should be 'void close(int forced) { ... }'.

> The second is that if I close the Mud Client I have without a quit command
> (I.e. I close the Mud Client program) then the DGD driver crashes.

I can't imagine that being the way things work, what version of DGD
are you running, on what platform, and what mud-client are you using?

> If I have no close command then it's fine ... *confused*

I think you mean function, not command.  Calling an undefined function
in an object will work no matter what parameters you tried to call it
with, but if the function exists and you're running with type-checking
mode 2 (I think) then it had better match with the parameters of the
function.

> Also I have a quit command that gives a goodbye message and then destructs
> the player object. However the message doesn't get shown ... The user is
> just disconnected ... What am I doing wrong?

It might help to do the disconnecting/destructing from a call_out() so
that the network-traffic is given a bit more time to be sent across.

> Lastly and most amusingly I wonder about rlimits. There doesn't seem to be
> any documentation on what you need in your mudlib to satisfy the DGD Driver
> and so we are working it out as we go along. Looking at other mudlibs and
> error messages. So far we haven't put any rlimit stuff anywhere. I don't
> really understand about stacks and ticks etc. at all.

You want rlimits() restrictions surrounding anything that doesn't have
a damn good reason for being allowed to finish 'no matter what'.

In general that means you override the standard call_out() so that the
function gets called with some rlimits() restriction, and similar bits
for open()/close()/receive_message()/etc functions, in short anything
that can start a new thread.

> Also security ... So far we don't really have any security checks on
> anything. What should we do about this?

Panic. ;-)  There are three levels of security as I see it:

1. Internal consistency, to make sure certain auto-object functions
   are only used by other auto-object code, for instance.

   For that type of situation previous_program() is really useful.

2. Filesystem security.  Do you want anyone or anything to be able to
   read/write to/from anywhere?

   To fix that you'll want to sit down and think about what the basic
   rules are, and how flexible you want to be in being able to 'grant'
   access to others, etc.  Overruling the basic file/directory-
   related kfuns is the next step once you've figured that out.

3. Cloning/compiling/destructing/inheriting/including.  Be careful who
   or what is allowed to use what other objects/files for this.

Good luck!

Erwin.
P.S: Did you just send the same message twice to the list?
-- 
Erwin Harte
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Par Winzell)
Date: Tue Dec 11 19:01:00 2001
Subject: [DGD] Compilation in System...

 > >I'd suggest putting it in /usr/World, if anywhere.  Be sure to create
 > >a pseudo-user for that, with 'grant World access'.
 > 
 > Didn't understand about pseudo-users, but now I have at least some grasp I 
 > see this is an excellent suggestion and I'll be using it.

The urge to make ~System as small as possible hit me later than I
would have wished, and I spent a lot of extra time moving things
out of ~System and into other directories. ~System should do almost
nothing but export the priviliges given to it by the kernel library
to the rest of the library, along with whatever restrictions you
want to add to the kernel library's.

Zell
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Stephen Schmidt)
Date: Mon May 19 23:15:01 2003
Subject: [DGD] Any advice?

On Tue, 20 May 2003, Ben Chambers wrote:
> While the challenge is part of my reason for doing this, I also feel that
> coding it myself will allow me to have more flexibility in how it is
> implemented and what exactly I want it to do and how it does it.  For
> example, I'm sure that because Melville was written from scratch, it is more
> like what you originally intended than if you had changed your ideas in
> order to allow the mudlib to be built on top of the kernel.

In truth, probably not, for two reasons. First, the kernel mudlib
is, if you build over it correctly, highly transparent to the
end goals. In terms of functionality, the kernel strikes me as
extremely flexible (though I'm not very familiar with it) and
I doubt it misses any features I would consider important. In
terms of elegance, that might not be true - the kernel has
features that I personally don't anticipate that I'd ever use,
and if I built something for myself from scratch, I could get
something that did less than the present kernel does, but did
everything I wanted, in less space (but much more programmer
time, of course).

Second, Melville was always oriented towards being a real simple
mudlib that would be familiar to someone who coded on MudOS or
other post-LP-2.4.5 drivers, but would let you start to get
into the internals of how DGD worked. In 1994 I think it did
a decent job of that. Nowadays the driver has moved so far
beyond Melville that it doesn't anymore. Today it's main
purpose is to give a familiar mudlib to someone who wants to
code more or less the same way they did in 1994 (which is
perhaps a dumb goal - why use DGD if you're not going to take
advantage of its power? - but the evidence shows that there
is a small but reliable market for it.) Both of those goals
could be carried out equally well in a mudlib that ran over
the kernel, and the "intro to DGD" one could, of course, be
carried out much better that way. Today I would guess that
Phantasmal, which does run over the kernel, is the right way
to go for someone relatively inexperienced in game design who
is interested in learning how to hack DGD. (Disclaimer: I know
Phantasmal only from discussion on this group, I may be totally
wrong about that.)

Steve
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Sun May 25 17:06:01 2003
Subject: [DGD] Rlimits weirdness...

Noah Gibbs wrote:

>   I'm seeing some odd behavior, and the Kernel Library
> definition of runtime_rlimits() in the Driver seems to
> be the cause.
>
>   An object in /usr/System is permitted to set
> maxticks to -1, which will remove all limits on ticks.
>  On the other hand, the same object isn't allowed to
> (for instance) set the limit to 250,000 repeatedly so
> that it can run many consecutive time-limited
> operations in the same thread.  Is this for security
> reasons and I'm just not getting it?

What you can't do is set a tick limit and then raise it.  Non-negative
tick limits can only be set when the current limit is -1 (infinite).
Once a limit is set, you are committed.  You can voluntarily lower it,
though.

>From your description, I think you should be setting the limit to
250,000 from where the current limit is infinite.  That'll allow you
to run the consecutive time-limited operations, as you describe.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Sun May 25 17:59:00 2003
Subject: [DGD] Rlimits weirdness...

Noah Gibbs wrote:

> --- "Felix A. Croes" wrote:
> > What you can't do is set a tick limit and then raise
> > it.  Non-negative
> > tick limits can only be set when the current limit
> > is -1 (infinite).
>
>   Hm.  So since call_out sets a non-negative tick
> limit, that means I can only lower it.  So I guess
> I'll need to make every upgraded() call its own
> call_out.  I was trying to avoid that.

You don't have to, as long as you go back to unlimited ticks first:

    rlimits (-1; 0) {
    rlimits (250000; 0) {
        task1();
    }
    rlimits (250000; 0) {
        task2();
    }
    /* ... */
    }

Or, even better:

    rlimits (-1; 0) {
    call_limited("task1");
    call_limited("task2");
    /* ... */
    }

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Sun May 25 18:23:00 2003
Subject: [DGD] Rlimits weirdness...

Noah Gibbs wrote:

>   I wanted to use call_limited, but it's static.  And
> rather than making every upgraded() call in every
> object use it manually (definite security hole, and
> inconvenient), I'd have to do the same thing from
> ObjectD, which can't get at some of the information it
> would need to do that.

Static functions can be called with call_limited().  Beyond that,
it is not clear to me exactly what you are trying to do, so it is
hard to offer advice.


>   So for the moment I just have two rlimits calls
> surrounding the call_other.  It looks silly, but it
> works.
>
>   I still find it very odd that the Kernel Library
> allows that, but disallows it with only one call.

It disallows raising the ticks limit, unless you are in "supervisor"
mode, i.e. running with no ticks limit.  Note that you cannot deal
with running out of ticks unless you do the dealing in code that has
no ticks limit (unless you make the whole operation atomic).

System objects are subject to the same restrictions on raising ticks
as other objects, to avoid accidents where the ticks limit is raised
forever in an infinite loop.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Ben Chambers)
Date: Mon May 19 23:23:01 2003
Subject: [DGD] Any advice?

----- Original Message -----
From: "Stephen Schmidt"
Sent: Tuesday, May 20, 2003 12:14 AM
Subject: Re: [DGD] Any advice?


> In truth, probably not, for two reasons. First, the kernel mudlib
> is, if you build over it correctly, highly transparent to the[...]

Learning to 'hack' DGD is most definitely not my only goal.  The one biggest
issue that I have with the kernel library is lack of documentation.  At
first glance I thought you would simply hack it to expand it.  Then when I
read stuff at Phantasmal's site (which is wonderful, I might add) it turns
out it looks in some directory in the user directory for extensions of these
objects that you can apparently drop in place yourself...
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Joshua P. Dady)
Date: Tue May 20 09:04:01 2003
Subject: [DGD] Any advice?

On Tuesday, May 20, 2003, at 07:19  AM, Felix A. Croes wrote:
> Having said that, some things in DGD are just mind-bogglingly complex.
> Take the string parser for instance -- it wasn't created to make
> parsing easy, it was created to make parsing really efficient.  The
> documentation for it I consider to be of good quality, but nevertheless
> making a good parser with it is as hard, or perhaps harder, than doing
> so in LPC.

At least in this case, I would expect that there are more than enough 
of us who understand that even writing good grammars to feed a generic 
parser are hard to write, and that there are good, dedicated texts on 
the matter, that it wouldn't be hard to explain it to anyone who didn't 
just understand.

As for the rest, I've often found situations where I would have to read 
the documentation _and_ the kernel mudlib's implementation of that 
section in order to really understand either.  This tends to be a 
two-edged sword, as it gives deeper insight as to why things are the 
way they are, but there are times when I just want to implement a 
feature without having a history lesson first.  8)

   -Josh
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Fri Feb  6 07:28:00 2004
Subject: [DGD] Melville under the Kernel Lib

Michael McKiel wrote:

> In the past archives, Steven has stated if he were to "do it again" he'd
> prolly choose to go on top of the kernel. Yet in quite a few posting's also
> has stated a 'distaste'(?) for the directory depth's of such. 
>
> Yet there doesn't seem to be any way to accomodate (what we've been calling)
> the Klib without "breaking" its directory requirements, and making it innured
> to any future patches from the experimental line of DGD.

If you want to re-do Melville on top of the kernel library, you have to
move files around.  That is a given.

As much as I like the kernel library, I'd like to see other mudlibs that
don't depend on it as well.  When everyone uses the kernel library, it
becomes harder for me to find bugs in DGD that the kernel library just
doesn't trigger.

Here is what the kernel library does, and what a modern mudlib should do:

 - Provide a layer of abstraction between the driver and the higher-level
   mudlib, so that the latter does not have to be aware of driver changes
   (other than changes in LPC itself).
 - A framework for persistence.  In DGD this is really simple, all you
   have to do is destroy all connection objects to let them "go linkdead",
   and to let objects that deal with absolute time know that there has
   been a reboot interval.  For example, a cron object which wants to
   run tasks at specific hours should reschedule its callouts after a
   reboot.
 - Separation of inheritables and other objects.  This is essential for
   any mud that wants to be able to entire inheritance trees of objects
   to the latest source version without destroying object state.  The
   kernel library does it through pathnames, but other methods are
   possible.
 - Separation through pathnames of clones and LWOs.  This is not
   essential, and the kernel library just does it for the sake of
   orthogonality.
 - Resource management and measurement.  This is absolutely essential for
   a persistent mud with guest programmers (think of LambdaMOO), and
   useful but not required in other cases.
 - File security and object security.  A requirement for muds with guest
   programmers, but hard to do right, and the kernel library's system
   may be too paranoid.
 - DGD/MP aware.  This is going to be important in the future.  A number
   of design issues are involved:

    - There have to be some kind of thread-local variables which are not
      held in any object, to handle things such as this_player().
    - It must be possible to start a callout without making a change to
      data in any object (this is why callouts are no longer a resource
      in the kernel library, since resources are tracked by a central
      object).
    - Large threads that modify many objects should, whenever possible,
      be broken up into smaller threads modifying fewer objects, using
      callouts.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Sat Feb 28 08:47:00 2004
Subject: [DGD] Kernel LIB's Invisible Callouts

Michael McKiel wrote:

> Well I've plowed thru the kernel, got it running under the following
> directory structure:
> /home   /include   /k
>[...]
> Ok enough preamble I suppose heh. The question being, why can't callouts be
> seen in kernel objects? ... Since if I want to mix kernel AND 'System' files
> it would seem to me nonVisible callouts would pose problems.

Who are you asking?  You changed it, it's your own mudlib now :)  If
you don't want callouts in /k objects to be invisible, make them
visible.  They are not visible in the kernel library because /kernel
objects are in a class of themselves, and should definitely <not> mix
with other objects.

You will have to do some conversion, though, since callouts in
kernel objects have a different argument format; they don't pass
through the _F_callout wrapper.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Michael McKiel)
Date: Sat Feb 28 09:36:01 2004
Subject: [DGD] Kernel LIB's Invisible Callouts

 --- "Felix A. Croes"
> 
> Who are you asking?  You changed it, it's your own mudlib now :)  If

*grin* well Yeah...but I couldn't proceed til I knew why all the Kernel's
call_out's were 'invisible' to status() ;)

> you don't want callouts in /k objects to be invisible, make them
> visible.  They are not visible in the kernel library because /kernel
> objects are in a class of themselves, and should definitely <not> mix
> with other objects.
> 
> You will have to do some conversion, though, since callouts in
> kernel objects have a different argument format; they don't pass
> through the _F_callout wrapper.


At first I didn't think that helped, but did some grep'ing and found that the
only kernel objects that do call_out's are rsrc.c and rsrcd.c, and they're
the only objects prevented from using the _F_callout wrapper:
due to (in auto.c's call_out) :
    if ( DIR_RSRC(oname) ) {
    /* direct callouts for resource management objects */
    return ::call_out(function, delay, args...);
    }

where, #define DIR_RSRC(f) sscanf(f, "/k/%*s/rsrc") != 0

So I don't have to prevent status() from displaying call_out's in all objects
that are/could be/ considered "Kernel" just 'DIR_RSRC' ones.

So...security here I come :), Thank ye.
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Wed Mar 24 05:25:01 2004
Subject: [DGD] Question about thread local storage

Steve Wooster wrote:

>      I've been looking through the kernel lib to see how it implemented 
> TLS... I know it has to do with modifying the array of arguments for the 
> second call that call_trace() returns, and that the kernel lib always 
> ensures the second call has extra arguments sent to it, but I'm a bit 
> confused with how it does that. When it comes to functions in the driver 
> object such as Initialize() or Restore(), I see it passing the necessary 
> array, but at first glance, I'm having trouble figuring out how it 
> allocates the memory in threads started by call_outs. Anybody know? Thanks. 
> My apologies if this could have been easily found with more searching.

All callouts are redirected to _F_callout() in the auto object.  This
calls _F_call_limited(), and that function then <replaces> the argument
array with the TLS array.  I could have handled this through another
function (say __F_callout) as it is done elsewhere, but _F_call_limited
is heavily involved in TLS manipulation anyhow.

By the way, the kernel library uses the _F_/_Q_ convention for functions
that have to be present in objects, but should not pollute the object
namespace.  If "_F_destruct" had been "destruct", then mudlibs based
on the kernel library could not have defined a destruct function
themselves (these _F_/_Q_ functions are not intended to be masked).

Regards,
Dworkin
</pre>
<hr>
<pre>
From: "Felix A. Croes"
Subject: Re: [DGD] Can't put a wiztool outside the /usr/System directory...
Date: Sun, 3 Oct 2004 22:22:06 +0200

Noah Gibbs wrote:

>[...]
>   Along the same lines, I tried putting a wiztool object in
> /usr/game/obj/wiztool.  I added a small function in the System
> directory to clone it with the wizard's identity, just like the regular
> /usr/System/obj/wiztool object works normally.
>
>   It *almost* works.  You can run a lot of the regular commands like
> "people" and "access global" successfully, because those check the
> owner of the object or the path of the program that's calling them. 
> But since a lot of the file operations check the creator of the object
> instead, you can't do things like grant somebody access (which creates
> a new directory for them) or determine what your own access (which
> involves enumerating a directory's entries).

When working on the kernel library, for a long time I had the idea
in mind that the wiztool should be inheritable by everyone.  It didn't
work out with the kernel library's security system, though.  So the
wiztool should be in /usr/System.  It's still a good idea to let people
write their own extensions for it, but those can't be added through
inheritance.

Regards,
Dworkin
</pre>
<hr>
<pre>
From: dgd at list.imaginary.com (Felix A. Croes)
Date: Fri Apr 12 08:20:02 2002
Subject: [DGD] Bug or feature?

Keith Dunwoody wrote:

> Suppose I have a file called ".c" in a directory
> /usr/blah, which I want to compile.  I tried using
> compile_object("/usr/blah/"), but instead of trying to
> compile "/usr/blah/.c" it tries to compile
> "/usr/blah.c".  Is this a bug or a feature?

It's a bug, but it's not going to be fixed, so some might call it a
feature :)

Regards,
Dworkin
</pre>
