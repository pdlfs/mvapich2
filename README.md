# mvapich2

The latest version of mvapich2 with legacy support for InfiniBand,
psm, etc. can be found here:

https://mvapich.cse.ohio-state.edu/downloads/

Unfortunately, as of this time, the most recent mvapich2 2.3.7-2:

https://mvapich.cse.ohio-state.edu/download/mvapich/mv2/mvapich2-2.3.7-2.tar.gz

has a couple of unaddressed bugs that may cause issues depending
on what you are trying to do and what system you are installing on.

The mvapich project does not appear to have a bug tracking system
(they suggest sending an email report to the mvapich-discuss email
list).

We are going to post mvapich2 bug info with fixes here in case
other find them useful.   We will also attach a tar file of
the patched mvapich2-2.3.7-2 sources to the releases page.

bugs:

## snprintf

On some systems mvapich2 binaries crash at init time with
error "*** buffer overflow detected ***" like this:

<pre>
nid000:testin/affinity % srun -n 1 /usr/local/mpi-psm/libexec/osu-micro-benchmarks/mpi/startup/osu_init
*** buffer overflow detected ***: terminated

osu_init:2521 terminated with signal 6 at PC=ad80d69eb2c SP=7ffd8ffa93e0.  Backtrace:
</pre>

This is due to bad code calling snprintf() in
src/mpid/ch3/channels/common/src/affinity/hwloc_bind.c.  e.g. when
mapping is a char array of size _POSIX2_LINE_MAX and you snprintf()
to it at a non-zero offset "j" you could have a buffer overflow
depending on the value of "j" ...

<pre>
j += snprintf (mapping+j, _POSIX2_LINE_MAX, ":");
</pre>

The patch snprintf.diff adds a new function that can safely be used
to snprintf at non-zero offsets and changes all the snprintf() calls
to use that instead.

## nemesis

The nemesis code is a TCP backend which can be used to compare mvapich2's
accelerated performance (e.g. with ibverbs or psm) to plain old TCP.
The nemesis code currently has some typos that prevent it from compiling.
The simple patch in nemesis.diff fixes allows nemesis code to be compiled
once again.

## hwloc-pmix

If you attempt to use mvapich2 with verbs and a version of slurm that
uses newer versions of hwloc/pmix it may crash at startup.  This is due to
mvapich2 linking with multiple conflicting versions of the hwloc library.

The mvapich2 code comes with old versions of hwloc embedded in its source
tree.  It supports the configure flags --with-hwloc=v1 and --with-hwloc=v2
that select which version of the hwloc library in the contrib directory
to compile and link with.  The default is --with-hwloc=v1.

To link mvapich2 with a slurm that uses pmix you must first compile
pmix and slurm.   Unfortunately, pmix requires linking with hwloc.
This can lead to the case where the pmix you compiled to build slurm
with links to a newer system-installed version of hwloc (prob hwloc v2).

Then when you compile mvapich2 it builds its own embedded internal
hwloc v1 to link to.   But mvapich2 also links to pmix, and pmix may
have been linked with a system-installed hwloc v2 library.

The result of this is that your mvapich2 build may be linked with
both hwloc v1 and hwloc v2 at the same time.   On our test build
on ubuntu24 this was the case and it resulted in a start up crash.

Specifically we found that when PMIx_Init() called hwloc_topology_init()
it got the v1 version from mvapich2.   But then when it used the hwloc
object created by that call to call hwloc_topology_set_io_types_filter()
it went to the v2 version of hwloc_topology_set_io_types_filter()
in the system library.  Since the internal format of the hwloc object
changed between v1 and the current v2 the hwloc_topology_set_io_types_filter()
function fails (thinking the object has already been loaded).  That
causes mvapich MPI_Init() to fail.

Our solution to this is to add "--with-hwloc=v2ext" to mvapich2
to tell it to NOT compile its internal hwloc library and instead
use the same system one that pmix is using.

The patch file hwloc-pmix.diff has the fix for this.  Unfortunately,
this patch changes autoconfig files.   In order for the changes
to take effect, you have to run "autogen.sh" to generate a new
"configure" script.   So if you apply hwloc-pmix.diff, run
"autogen.sh" after patching.
