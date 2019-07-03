[About the author and ways to get in touch](https://docs.google.com/document/d/1TmA8cXRxb_9lDnCREfYuIeGpJy9sMsXSpFVMK-0PwNo/edit).  

Miscellaneous ideas and opinions that I would like to share and that are outside of [The Code-Reviewer's Notes](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md) topics.

----
+ [The Build System of My Dream](#the-build-system-of-my-dream)
  + [Why I'm Writing This](#why-im-writing-this)
  + [Exclude the Never Used Code and Minimize the Used One](#exclude-the-never-used-code-and-minimize-the-used-one)
  + [Build the Most Failing Files First](#build-the-most-failing-files-first)
    + [Restructure Your Sources](#restructure-your-sources)
  + [Prefer the Multithreaded Build Utilities](#prefer-the-multithreaded-build-utilities)
  + [Avoid Copying or Prefer the Links](#avoid-copying-or-prefer-the-links)
  + [Know About the `ccache`](#know-about-the-ccache)
    + [How `ccache` Works](#how-ccache-works)
  + [Optimizing the Code Repository Interactions](#optimizing-the-code-repository-interactions)
    + [Store Non-Source Files Separately](#store-non-source-files-separately)
    + [Know About Shallow Cloning](#know-about-shallow-cloning)
    + [Avoid Cloning](#avoid-cloning)
    + [Know About a Single Local Repository and Multiple Working Directories](#know-about-a-single-local-repository-and-multiple-working-directories)
    + [Build Differently the User Branches And the Main Branch](#build-differently-the-user-branches-and-the-main-branch)
  + [Optimizing the Build Servers](#optimizing-the-build-servers)
    + [Optimizing the Sources for the Build Servers](#optimizing-the-sources-for-the-build-servers)
      + [Optimizing the Floating Licenses Use](#optimizing-the-floating-licenses-use)
    + [Know the Limitations of Virtual Machines](#know-the-limitations-of-virtual-machines)
    + [Know About the `chroot`](#know-about-the-chroot)
    + [Know About the SSD Drives](#know-about-the-ssd-drives)
    + [Know About the RAM Drives](#know-about-the-ram-drives)
      + [Problems With the RAM Drives](#problems-with-the-ram-drives)
  + [See Also](#see-also)
----
+ [Bug Prevention and Code Reviewing vs. Testing With Bug Fixing](#bug-prevention-and-code-reviewing-vs-testing-with-bug-fixing)
  + [The Life Cycle of a Bug](#the-life-cycle-of-a-bug)
  + [The Bug Prevention](#the-bug-prevention)
  + [Summary](#summary)
  + [What to Remember](#what-to-remember)
  + [See Also](#see-also-1)
----

# The Build System of My Dream

(Draft)  
Here I share my ideas about how one can accelerate the source code build process a hundred times.

## Why I'm Writing This

We, software developers, spend significant part of our life waiting for the build to complete. Especially in the projects with, what I would call _rough build granularity_, i.e. one makes a small change but has to recompile lots of files (the build scripts do not let rebuild just that file). Or in the projects with lots of dependencies one changes a file on which lots of other files depend.

We _build_ our changes locally, debug them, fix, _rebuild_. When we are satisfied with our changes we commit them. If we use GIT then we commit to the local repository. To make sure that we did not forget to commit any changes or new files, we check-out from the local repository to a clean directory and _rebuild_, then we push our branch to the remote repository, and submit for code review. As a result of the code review we repeat a few iterations of code changes and _rebuilds_.  

After the code review, before we can merge our branch to the main branch, we have to (or are expected to) wait until the continuous integration (CI) system fully _rebuilds_ our branch (to make sure that our branch has no breaking changes). This remote full _rebuild_ is done upon every push of the code review updates.

After we merge our branch to the main branch the CI system _rebuilds_ the result of the merge and we can get the failure reports, i.e. we will need to fix the build breaks and repeat the iteration. How many _rebuilds_ did you notice?

In the build system of more than one of my employers in a row I see the things that run inefficiently, that can be accelerated. Here I share my experience and I hope that all my future potential employers will read through this, implement everything that is applicable to them, and when I come to them I will not have to spend time for explaining all this, I will just immediately dive into and enjoy the fastest possible build system, and concentrate on the creative parts of my job.
Let's go.

## Exclude the Never Used Code and Minimize the Used One

In many code bases I see the never used code.

The never used local variables are easily caught by using certain compiler flags. The situation is not as good with global and member variables. With variables to which the value is assigned but is never used (the variables that are never read from). The situation is even worse with data types, constants, macros, `#include` directives, and even the whole files that are never compiled, even worse the whole binaries (libraries, executables) that are built but are never used.  
Whenever we make code changes we have to make sure that the changes are consistent (e.g. if we rename a variable then we have to make sure it is renamed _everywhere_). While making such changes we sometimes dive into the never used code, spend lots of time in there before we understand that the code is dead.  
With years of experience I understood that every character we type has a consequence for many years to go. Every character written distracts us if it becomes a dead code, slows down the migration to the new compiler version and porting to the new platform, etc.
We should type as little code as possible.

Unfortunately I don't know the tools that can effectively automate the detection of those never used and non-minimal code fragments. If you know one then please share and let me know (there should be a link in the top of this page about how to get in touch with the author) and I will add here a link to your post. Or feel free to contribute to this page.
If you develop a compiler then you can add the dead code detection at a file level. If you develop a linker then you can add the detection at a binary level (e.g. the variable/constant/type is never used in a binary and is not accessible from outside of the binary).
If you develop a static analyzer then you can add at both levels.

As long as there are no automatic tools to help, bear in mind this philosophy. Whenever you notice the never used code, either remove it (or comment it out) or explicitly mark it as never used, e.g. place the never compiled files/directories to a separate directory named e.g. "never_compiled" or "removal_candidates", etc.

## Build the Most Failing Files First

The mature code bases typically have files that are changed relatively rarely, and the ones that are changes frequently. The frequently changed files are more likely to make the build fail. If the build fails then it is more efficient to find out about the failure _early_. For that build the most failing files first.  
Since the most changes go to the newly added files, add the new files to the _beginning_ of the list of files to compile, add new projects to the _beginning_ of the list of projects to build, etc. E.g.
```makefile
FILES_TO_BUILD = 
  file_added_later.cpp \
  file_added_earlier.cpp
```
Sometimes compiling the most failing files first requires you to

### Restructure Your Sources

I dealt with a build system where the sources consisted of two parts: the x86 utilities, and the ARM code. The parts were taking about equal time to build. The x86 utilities were built first, because it was believed that they generate the header files used by the ARM build.
A brief analysis has shown that only one of the x86 utilities was generating the ARM headers. All the rest of utilities were used by the build process _after_ the ARM build.

The x86 utilities were updated may be once in 6 months, whereas the ARM sources were updated multiple times per day. As you can guess during the full rebuild many build failures were taking too long to happen.

To improve things the build process needed to be restructured such that only one of the x86 utilities was built first, then ARM code, and then the rest of the x86 utilities. Such a restructuring would accelerate the build failure detection a few times.

## Prefer the Multithreaded Build Utilities

Some build utilities are not multithreaded, e.g. nmake 9.00.30729.207.
Some others have the multithreading flags, e.g.
* the `-j` flag of [GNU make](http://man7.org/linux/man-pages/man1/make.1.html) ([Note that this option is ignored on MS-DOS](https://www.gnu.org/software/make/manual/make.pdf), I haven't checked against Windows yet);
* the `-parallel` flag of the `iarbuild.exe` utility that can be used to build in command line the projects of IAR Embedded Workbench for ARM 8.32.1 (see the PDF installed by default to `C:\Program Files (x86)\IAR Systems\Embedded Workbench 8.2\arm\doc\EWARM_IDEGuide.ENU.pdf`), I haven't tried yet;
* the `/maxcpucount` flag of the `MSBuild.exe` utility that can be used to build in command line the Microsoft Visual C++ projects, I haven't tried yet.

The multithreaded build is a few times faster than the single-threaded one. However it has its caveats. In particular:
1. The output of multiple files built in parallel is interleaved. E.g. in the console you can see
```
Building B.cpp
Building C.cpp
Building A.cpp
```
And then the compilation of `B.cpp` may print warnings or errors. The unprepared developer may believe that the warnings or errors relate to `A.cpp`, and waste quite a bit of time.

2. The compilation steps can be executed in a wrong order. E.g. the binary depends on the object file, i.e. the binary needs to be linked after the object file has been compiled. But during the parallel build the linker sometimes gets kicked off earlier than the compiler finishes with the object file. In that case the linking fails with the error message that is not always evident. This issue arose a number of times with the `gcc` toolset used with `make -j`.  

One of the reasons of such a wrong order may be the following. The compiler writes the last chunk of data to the object file, that chunk of data is saved in the output buffer (not yet flushed onto the disk). The compiler exits and returns the control to the build system. The build system kicks off the linker. The linker sees that the object file is either absent or is still blocked or is incomplete, and reports failure. Then the flushing of the compiler's output buffer is done and the object file becomes complete.
If you are the compiler or other toolset developer then make sure this situation does not take place with your tools.
If you are a _user_ of the toolset then the simple workaround may be multiple launch of the build command, something like `./build.sh || ./build.sh` (if the first `./build.sh` fails then one more is run). Multiple multithreaded runs still execute a few times faster than one single-threaded build.

## Avoid Copying or Prefer the Links

If during the build process your scripts copy the files then in your build system something is likely designed in a suboptimal way. In most cases copying can be avoided. Take a closer look at your build steps.

If the copying cannot be avoided then prefer the symbolic or hard links.

The links are part of the file system. They are more efficient than copying. The principle is the same as, in programming, instead of creating multiple copies of the same data you create multiple pointers to it.
See the documentation for the links
* Linux: http://man7.org/linux/man-pages/man1/ln.1.html
* Windows: `help mklink` or `mklink /?`.

I didn't use the hard links (as opposed to symbolic links) enough to talk about them. In the text below whenever I mention "link" I mean symbolic link by default.

The links also provide grounds for a very efficient technology used by `ccache` described below.

## Know About the `ccache`

(I'm not sure if this or similar tool exists for Windows)  
[ccache](https://ccache.dev/) is a command line utility that can get control instead of the compiler, invoke the compiler, cache the compiler's output - the object file, and during all the subsequent attempts to build the same file with the same compiler with the same compiler arguments, the cached object file is immediately provided by the `ccache` utility instead of invoking the compiler.

### How `ccache` Works
In the PATH environment variable there are multiple entries. When you (in command line) or any application (e.g. the IDE) launches the compiler with the command `gcc`, that `gcc` is searched for among the PATH entries. One of the entries (I assume not the first-most entry in the PATH) contains the path to `gcc`.  
If _earlier_ in the PATH we create a link named `gcc` then that link will be found earlier than the compiler executable. And that link will be launched instead of the compiler. The link `gcc` can be pointing to the `ccache` executable. And we hit a rather curious situation.  
We try to launch the file with _one_ name (`gcc`) but the control is received by the file with the _different_ name (`ccache`). Inside of `ccache` the command line parameter number zero is curious. In UNIX scripts this parameter is named `$0`, in Windows scripts it's named `%0`. This parameter is intended to contain the command (optional path and mandatory name) used to invoke the executable or script. In C/C++ the parameter can be named `argv[0]` in the fragment like this:
```c++
int main(int argc,           // The number of command line arguments.
         const char *argv[]) // The array of pointers to the command line arguments.
{
    .. argv[0] .. // The parameter `argv[0]` is the pointer to 
                  // the `[path/]file` used to launch the executable.
}
```
Inside of `ccache` the `argv[0]` is still `gcc`. Thus `ccache` can understand that it has been invoked instead of `gcc` compiler. The `ccache` parses the rest of the command line arguments, and if it detects that the source file is compiled for the first time, then the `ccache` invokes the compiler `gcc` (found later in the PATH), passes to it all the command line arguments, caches the `gcc`'s output - the object file, and the info about the original source file compiled and compilation arguments. If the `ccache` is invoked again with all the same arguments, and the file to be compiled is still the same (even if its timestamp is different), then instead of calling the compiler the `ccache` copies the cached object file to where it is expected to appear. 

With the `ccache` turned on the first-most compilation is promised to be about 20% slower in average (in my case I got an impression of 3 - 5% slowdown). All the subsequent compilations are a few times faster.

You can clear the file cache of the `ccache`, change its location, measure the needed size and change it, etc. Play a bit with the `ccache`: `ccache --help`.

As for Windows, I believe it is quite possible to compile `ccache` to run on Windows (in [MinGW](https://mingw-w64.org/doku.php) or other UNIX-like shells, e.g. [Cygwin](https://www.cygwin.com/), I never used Cygwin). However I don't expect the `ccache` to understand how to work with the compilers mostly executed on Windows such as MS VC++, IAR  (I don't expect the `ccache` to understand the compiler flags, output, etc.).

Thanks to [Volodymyr (Vova) Kuznetsov](https://www.linkedin.com/in/volodymyrkuznetsov/) for letting me know about the `ccache` (in 2005 - 2006).

## Optimizing the Code Repository Interactions

This section is mostly applicable to those who use GIT. Some sections can be applicable to the other version control systems.

### Store Non-Source Files Separately

I regularly see source code repositories that contain compiled library binaries, PDF files, executable utilities, and other large files that do not participate in the build process. I even saw a repository containing the Eclipse installer, and nobody cared that full cloning was taking 20 minutes.
Remember, every file in the repository slows down all the interactions with the remote repository. Most interactions with the remote repository in my case take place on the continuous integration (CI) build servers. As mentioned earlier the build servers rebuild very many times and they always get the full updates from the remote repository.
I would recommend the following:
* Keep in your primary repository only the files 
that participate in the build process, 
AND that you update frequently.
Customize your build servers to fully get and rebuild this primary repository only (during the regular CI builds).
* Keep the third-party library _source code_ and all the source code that you update _rarely_ in the secondary repository. Customize your build servers to rebuild secondary repository upon explicit request only (exclude this secondary repository from the regular CI builds). 
If the third-party library is provided in the form of the compiled binaries then those binaries can also be stored on the secondary server. However the secondary server still should contain only the files that _participate in the build process_.
* Keep all other files that do not participate in the build process either in a separate repository or on a file share.

### Know About Shallow Cloning

When the build servers rebuild the code they need one particular commit of one particular branch. If your build servers do a full cloning during every rebuild then you need to be aware of the `--single-branch`, `--depth`, `--branch`, `--shallow-submodules` flags of the [`git clone`](https://git-scm.com/docs/git-clone).

### Avoid Cloning

For me cloning always worked too long. I found that local copying of a directory containing the local repository `.git` (with subsequent `git fetch` optionally followed with `git checkout` and/or `git merge` (some prefer `git pull`)) is much faster and works as good. The remote CI servers can do the same.
Thus whenever I need to work on multiple branches in parallel I just keep multiple local repositories. This is rather disk-space consuming approach. If the disk-space is an issue, or the history of the branch commits does not matter, and you don't need to keep track of the changes you make (all of which were the case for the build servers I dealt with) then one can make another step in optimization efforts and use, again, the symbolic links instead of copying. For that

### Know About a Single Local Repository and Multiple Working Directories

Let's say a colleague of mine asked me to investigate (but not fix) a bug in `branchA` and a different bug in `branchB`. I can clone the repository to the directory `shared_repo`, the local repository will be in the directory `shared_repo/.git`. Then next to the directory `shared_repo` I can create directories `bugA` and `bugB`, inside of each I can create a symbolic link named `.git` pointing to `../shared_repo/.git`. Then I can enter the directory `bugA` and do `git checkout --force branchA` (written from memory, TODO: double-check). Thus inside of directory `bugA` I will get a clean working directory of the GIT branch `branchA`. I do similar thing for `bugB`. And then work in parallel on 2 bugs in 2 branches.
The main disadvantage of this approach is that, since multiple working directories share the single local repository, the change to the local repository (e.g. a checkout of GIT branch `branchB`) made in one working directory (e.g. in directory `bugB`) affects all other working directories (e.g. directory `bugA`). In particular all other working directories become inconsistent with the local repository (in directory `bugA` upon `git branch` the local repository says that the `branchB` is checked out, but the directory has the sources of `branchA`; `git diff` will be useless, etc.). To summarize, in such an environment it is very tricky to keep track of the changes you make. To work around, certain bookkeeping needs to be done to remember which directory has which branch checked-out, and which GIT commands need to be executed to synchronize the local repo and the working directory.

All those issues are not the case for the build server. The build server just needs to get the files of a particular branch, build them, proceed to the next branch. One local repository is enough for all the builds the build server ever does. The disk space is used much more efficiently (compared to multiple local clones), which means that the longer history of the build artifacts can be kept. And no need to do a lengthy `git clone`.

Here some of you will strongly disagree with me and exclaim: "No. We need to make sure that all the operations including the full cloning, checkout, etc. always work for our repository". And I will reply:

### Build Differently the User Branches And the Main Branch

I'll remind that (in my case) the continuous integration (CI) system rebuilds the main branch after every merge of the user branch (after completion of the code-review/pull-request).

(In some systems the rebuild takes place even after the fast-forward merge (when the main branch "pointer" is redirected from an earlier commit to the later commit in the linear history, and the later commit has already been built as part of the pull-request). But this is not the point of this section, although is another item that can be safely optimized out and unload the build servers)

For every merge to the main branch there are multiple updates to the user branches. Some of those updates are part of the pull-request/code review, some are just back-up updates by the user in his/her private branch in the remote repository. I saw a system where even those back-up updates (that are not part of the pull-request) were fully rebuilt automatically (applying an extra burden on the build servers - another item for you to consider).

To keep the story short, on the build servers
* Apply the full rebuild for the main branch only. For all other branches optimize the build as much as you can.
* For the private branches that are not part of the pull-request build upon an explicit request only.

## Optimizing the Build Servers

### Optimizing the Sources for the Build Servers

(I will remind some of the mentioned above)
I worked with a project where the sources consisted of two parts: x86 utilities and ARM. The ARM code was built for a few generations of the device (e.g. GenA, GenB, GenC). Each generation had Debug and Release configuration. In order to build the ARM code we needed the x86 utilities to generate some of the headers. That is why the x86 utilities were built first. The x86 utilities also had the Debug and Release configurations. The whole picture resulted in a set of 6 flavors: 
GenA Debug, GenA Release;
GenB Debug, GenB Release;
GenC Debug, GenC Release.
The build system the was building the flavors in parallel. It was assigning the flavor `GenA Debug` to one build agent, `GenA Release` to another, and so on. Totally 6 build agents.

When I got a chance to look at the build process I noticed a number of curious things.

All 6 build agents in parallel were getting from the GIT repository the same commit (rather lengthy operation).
All the 3 Debug flavors were building the same Debug version of the x86 utilities.
All the 3 Release flavors were building the same Release version of the x86 utilities.

I believe you can guess the conclusion. If we delay the parallelizing then we can do certain things more efficiently. In particular we can get from the GIT repository the sources to one build machine only. Then copy those sources to one more build machine (or share over the network or over the shared folder if both build agents run on the same physical machine). Then on one build agent to build the Debug version of x86 utilities, on the other agent the Release version. Then the Debug agent (or the build system) could kick off 2 more build agents, share with them the Debug version of the x86 utilities, and build the 3 Debug flavors - GenA Debug, GenB Debug, and GenC Debug - of the ARM code on 3 Debug agents. Same could happen to the Release build. The Release x86 utilities are built on a single machine and then shared by the 3 agents building GenA Release, GenB Release, and GenC Release flavors.
Such an approach uses the build agents more efficiently.

#### Optimizing the Floating Licenses Use

In the same story described above there was another moment requiring attention. In order to build the ARM code the compiler license was required. We had a number of floating licenses used both by the developers and the build servers. If a machine compiles the ARM code then the floating license is assigned to that machine for 30 minutes (that's how the ARM toolchain vendor designed the licenses, we were not able to change that). The build of a flavor on the build agent was taking about 15 minutes, of which for a few minutes the GIT repository was accessed, then the x86 code was built, and then for about 5 minutes the ARM code was built.

I guess you see the problem. The ARM code is built for 5 minutes, but for 30 minutes after that the license stays occupied by the build agent. No other agent or developer can use that floating license to build the ARM code. If a new build task is assigned to the agent holding a license then that agent again will be accessing the GIT repository for a few minutes, then building the x86 code for a few minutes. I.e. before it proceeds to actually building the ARM code it will be holding the license, not using it (to build the ARM code), and not letting any other machine to use it. Very inefficient.

How to optimize? I believe you can come up with better solutions than I suggest below.  
We could separate the x86 build and ARM build. Build the ARM code on a specific set of agents only. Build the x86 code preferably on the agents other than the ARM agents. Thus whenever ARM build task arises it is assigned to the ARM agent (still holding a license from the previous ARM build). And whenever the GIT-access task or x86 task arises it is assigned to a non-ARM agent (thus keeping the ARM agents available for the ARM tasks). If there are no non-ARM agents available then the GIT-access task or x86 task is assigned to the ARM agent.

This results in a more complex build system that solves a larger number of smaller tasks, some of which are agent-specific.

This section is just a shallow example of what can be done. If you look deeper in your case then you can come up with a more optimized solution.

### Know the Limitations of Virtual Machines

I remember a continuous integration (CI) system with 2 - 4 physical servers, on each a few virtual machines were running. Each virtual machine was implementing a build agent of the continuous integration (CI) system. 

Each physical server had 8 (or more) CPU cores. 2 CPU cores were assigned to each virtual machine (i.e. each virtual machine was using 2 physical CPU cores).

Often there were low load periods when only one build job needed to be done. That job had 2 flavors (x86 and ARM), and those were assigned to 2 virtual machines. And everybody believed that the system was working fine. Nobody cared that the each flavor build was taking 40 minutes.

The key fragment in all this story is this: "Each virtual machine was using 2 physical CPU cores". I.e. totally 4 CPU cores were building, all the rest of the CPU cores were _doing nothing_.

Knowing what is below in this post I believe that one physical machine could be more performant than all those 2 - 4 servers each having a few virtual machines (even during the highest load periods).

### Know About the `chroot`.

[`chroot`](https://help.ubuntu.com/community/BasicChroot) is a technology (available at least in Linux) that allows to install one more operating system in a particular directory. E.g. on Ubuntu Linux you can create a directory and install in there the same or earlier version of Ubuntu Linux. You can log in to that guest operating system and work in there. Both the host and guest operating system
* will be running _in parallel_ (you don't have to choose during the boot which operating system to load)
* and _natively_ (as opposed to the virtual machines). It is like a virtual machine but running faster and using all the CPU cores.

What `chroot` gave me.
If in your development you have to use the [board support package](https://en.wikipedia.org/wiki/Board_support_package) provided by a vendor and supported on the old operating system only (in my case it was supported on Ubuntu Linux 12.04) then you have to use old Ubuntu Linux 12.04 on your development machine. You have to use old browsers (not supported by GitHub, banking apps), etc. and your life is gloomy.

With `chroot` technology you can install latest version of Ubuntu Linux and in a specific directory install the old Ubuntu Linux 12.04, log in to it and do all your development in there. Outside of development you enjoy all the latest feature of the latest OS.

Getting some experience I saw a number of 

Side Benefits of `chroot`.

The guest and host operating system _can share directories_. It means that a number of tools can be installed on the host, and shared by all the guests. It means that the file cache of the [`ccache`] (TODO: link to the earlier section) can also be shared between all the multiple guests of the same host. If each guest implements a separate agent of the CI build system then the build file cache gathered in one guest will be reused by the other guests. Since the `ccache`'s file cache is shared (and only a single copy is stored on the host) we can make the file cache large enough to store all the flavors. And whichever flavor build is assigned to the build agent the build process will still be using the file cache of the previous builds (by the same or other guests).

### Know About the SSD Drives

From the point of view of this post the [SSD Drives](https://en.wikipedia.org/wiki/Solid-state_drive) can be simplified to just faster hard drives, having the same hardware interface to the motherboard. As I know they are faster from a few times to a few dozens of times depending on the technology and what they are compared with.

I have never worked with SSD drives but my 16-year-old son has installed Windows 10 onto the SSD drive on his PC and the OS was booting in 20 seconds. SSD is a technology available for ordinary people. 

### Know About the RAM Drives

Would be right to say that I have no experience with the RAM drives but based on what I know and saw the RAM drives seem the most promising and culmination point of the build process acceleration.

A part of the RAM can be allocated and mapped as a directory of the file system. If you copy your sources to that directory, your sources will be loaded to the RAM (under the hood). If you build in that directory then all your sources will be built in RAM. All the intermediate and output files will be saved in RAM. No any access to the (slow) hard drives at all. Such a build promises to be tens of times (and may be more than hundred times) faster than the build on the typical hard drive.

After the build the artifacts are saved on the hard drive. All the rest can be quickly erased and the space reused.

#### Problems With the RAM Drives

The main problem with the RAM drives is that they typically have relatively low capacity. Not any project will fit on the RAM drive. However if you apply all the items above, minimize your sources, increase reusability, etc. then this problem can be not that bad.

Then you can break up your sources into independent pieces, copy them to the RAM drive and build piece by piece. You can also restructure the build scripts, e.g. delete the object files as soon as the binary is generated from them (and they are not needed any more). After which the space can be reused by the object files for another binary. And even that is not always needed.

In my last job I was dealing with the sources that after a full build were occupying a few gigabytes per flavor. I.e. if I got a server with 64GB of RAM, I could allocate 32GB for a RAM drive, keep the `ccache`'s file cache (for all the flavors) on the RAM drive, build the flavors on the RAM drive one by one, and all that could work for 2 `chroot`s in parallel (sharing the `ccache`'s file cache). I suspect that the performance of such a machine could be comparable to tens of physical machines building on a hard drive.

Unfortunately currently I cannot provide good links about the RAM drives. For me this area is still to investigate.

## See Also
* Phillip Khandeliants. [Speeding up the Build of C and C++ Projects](https://www.viva64.com/en/b/0549/) (+[RU](https://www.viva64.com/ru/b/0549/)).
* [C++ on Sea 2019] Viktor Kirilov. [The Hitchhiker's Guide to Faster Builds](https://www.youtube.com/watch?v=anbOy47fBYI) (video, 1:26:30).


# Bug Prevention and Code Reviewing vs. Testing With Bug Fixing

From time to time I hear something like this:
* "Let’s not spend too much time for code-reviewing, let’s focus on bug fixing". 
* "Migrate to a newer compiler? Not this year, we have too much bugs to fix by the year end".
* "Adding (in addition to Debug and Release) the extra build configurations (e.g. Maximum optimization for speed, Maximum optimization for size), and running the tests against them, is a huge overkill. That’s not what we want and/or have time for. What we need is to write more and better tests".
* "Adding extra compilers? Spending time for changing our code to compile successfully on other compilers? Just for the sake of extra compiling (and never use the generated binaries)? We definitely have no resources for that, did you see how many bugs are on our list?"

Even more sadly, at times I see in the people’s eyes (and never hear that explicitly): "We don’t want _new_ bugs to be found/disclosed/filed, we want to get rid of the _known_ ones (and to make an impression that very few bugs are found and we fix them efficiently, and that the code is fine/safe). Please stop your bug search. Please let's conclude the code review and move on. Please don’t remove the suppression of the warnings. Please don’t ask us to turn that static analyzer on".

Here I make a deep sigh.

On the one hand I feel a common misconception about what is more important and efficient for the project and how it affects the bugs on the people’s list.  
On the other hand (and this is a larger and more serious issue) I see that sometimes people are awfully afraid of bugs. Mostly they have strong objective reasons for their fear such as  
the annual bonus for a large team depends on the number of bugs opened and resolved,  
the audit and certification can fail or get delayed for the large company or product because of the extra bugs found,  
a number of clients are using the product and rely on it, new bugs reported can result in a loss of clients, reputation damage.

Here I’m trying to settle the misconception and to dissipate the fear. For that let’s consider 

## The Life Cycle of a Bug

The bug typically appears when the programmer writes the code. Many bugs die when the programmer re-reads one’s own code, and when the programmer drives the code through the preprocessor + compiler + linker, followed by the debugger (the programmer makes sure that the code works as expected). Then optionally either the same programmer or someone else may write and run the test for the added change (in the Test-Driven Development the tests are written _before_ changing the code).  
When the programmer is satisfied with the change he or she passes the code to the code review. In general the team of code-reviewers possesses broader knowledge than a single programmer and the team helps the programmer to sweep out another group of bugs.  
In parallel to the code review, typically a build and test-run for multiple configurations takes place on the build servers as a part of the continuous integration process, which helps to eliminate another group of bugs.  

Once the code review is over and the build succeeds, the change is merged to the code repository. However some bugs still survive. Moreover, a new group of bugs can appear here. As has been heard from [Thomas McGuire](https://cppcon2017.sched.com/thomas.mcguire2) at [one of the trainings](https://cppcon2017.sched.com/event/BhdM/debugging-and-profiling-c-code-on-linux): "The most sophisticated bugs are a result of the joined efforts". Two correct independent changes (made by two independent programmers), when they meet in the code repository, can contradict to each other and create new bugs, often rather complex.  

Finally there are bugs in the repository.  
Not all of them get control immediately. Not all of them after getting control show themselves right away. Often they stay inactive ("kind"), at times they form the category of what I call _risks_ rather than bugs, e.g. the code works fine as long as certain data type has certain size (for instance `sizeof(wchar_t)` is `2`). And once the code is migrated to a newer compiler (or is ported to a different platform, etc.) the bug causes the difference in behavior, and that difference can be detected much later during the run, and result in a long investigation. Or the bug can cause an undefined behavior, which can result in many different weird things. In the best case it is a reliable crash. In more complicated case it is different behavior during different runs (of the same binary), on different machines, with different compiler arguments, with different optimization levels, with different compilers. In the most complicated and severe cases it is a corruption or loss of data upon coincidence of a number of rare events, e.g. a corruption (and full loss) of a database file once in 6 months, plus-minus 4 months (and nothing crashes, justs starts failing).

Eventually the bugs that get control and show themselves, get noticed/detected by the test infrastructure. They are chased down and fixed. However the fix at times is very expensive. Because e.g. it is difficult to reproduce the bug (in order to start working on it), it can be difficult to chase the bug down during the debugging (especially in multi-threaded/multi-process/multi-machine distributed environment), it can be difficult to not break the other things with the fix, the fix can require running it through the debugger, writing extra tests, running the tests locally, code review, continuous integration build, another round of the full test run (followed by a detection of a minor typo, followed by a start over).

Finally the _reported_ bugs get fixed. And the bugs' life cycle is over.  
The key word here is "*reported*". This means that only the bugs that _get the control_, and only those that _show themselves_ (the "angry" ones), and only those that are _caught_ by the test infrastructure, get fixed. My experience tells me that it is a small amount compared to the number of bugs that _stay_ in the code repository after the run through the test infrastructure. And those that are fixed are fixed at a very high price, because of the reasons listed above, and, more importantly, because the quality of the bug reports by the test infrastructure is very low. The test infrastructure mostly reports the bugs like this: "At the moment M we expected to see A but instead we saw B". Nearly never it reports the bugs like this: "In the source file F, line L, we see `Printers` but there should be `Printer&` (the last character should be `&` instead of `s`)".  
The test infrastructure fights only the active ("angry") bugs. It fights with the bugs in the second half of the bug's life cycle. The bugs are always ahead of the test infrastructure, and the infrastructure always catches them up.

As opposed to the test infrastructure, it is order of magnitude more efficient  
to fight all the bugs at once, including those that currently don't get control, or don't show themselves,  
to fight with the bugs in the beginning of their life cycle, starting with the first compilation,  
to get the bug reports that provide the exact file path, line number, and the matter of the bug.  
And that's what is done by what I call

## The Bug Prevention
With _Bug Prevention_ I mostly mean creating an infrastructure for and running the code through  
* as many [static](https://en.wikipedia.org/wiki/List_of_tools_for_static_code_analysis) and [dynamic](https://en.wikipedia.org/wiki/Dynamic_program_analysis#Memory_error_detection) analyzers as reasonable,  
* as many different compilers as reasonable,  
* as _new_ compilers as possible ([Mateusz Pusz](https://cppnow2019.sched.com/speaker/mateusz.pusz) said that if the code is old and requires the older standard of the language, then the new compiler can be used in the compatibility mode with the old standard, but the new compiler will still report more issues compared to the old compiler, the new compiler will partially act as a static code analyzer),  
* as high warning level as possible,  
* as many extreme configurations as reasonable (no optimization, highest optimization for speed, highest optimization for size, etc.),  

and adding all that to the continuous integration process.

After or in parallel to all those automated tools of bug prevention, the code still needs to be run through a _thorough code review_. The code review catches the bugs that no other tool and infrastructure can catch. And in my personal case reviewing the code is more efficient bug fight than working as a part of the test infrastructure (writing and running tests, fixing the bugs reported by the infrastructure). This can be opposite for other people, depends on the passion and prior experience.

## Summary  
Here I would like to remind a phrase from the beginning of this post. But this time I will reword a little bit: 

> "Let’s not spend too much time for code-reviewing and bug prevention, let’s focus on bug fixing".

This is one of the most absurd phrases that I continue to hear regularly. It sounds to me similar to this: "(The side of our ship is leaking) Let's not spend too much time for patching the side, let's focus on getting the water out of the ship (with buckets and bathtubs)".  
I apologize if my opinion hurts you.

## What to Remember
* Having equal resources, { the bug prevention and code reviewing } _alone_ is order of magnitude more efficient than the { whole test infrastructure and bug fixing } _alone_. Ideally both should be present. But if the team has limited resources then the bug prevention and code reviewing should have higher priority.
* Some work environments induce the code owners to dislike the bug preventers and productive code reviewers. It is inefficient or even risky if the code owner manages the bug preventer or manages a more efficient code reviewer.
* Bug prevention is not certain type of testing or debugging. It is a separate activity.

## See Also
* [The Code-Reviewer's Notes](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md).
* [The Ultimate Question of Programming, Refactoring, and Everything](https://www.viva64.com/en/b/0391/) (+[RU](https://www.viva64.com/ru/b/0391/)).
* [A post about static code analysis for project managers, not recommended for the programmers](https://www.viva64.com/en/b/0498/) (+[RU](https://www.viva64.com/ru/b/0498/)).
* [Philosophy of Static Code Analysis: We Have 100 Developers, the Analyzer Found Few Bugs, Is Analyzer Useless?](https://www.viva64.com/en/b/0534/) (+[RU](https://www.viva64.com/ru/b/0534/))
* [PVS-Studio ROI](https://www.viva64.com/en/b/0606/) (+[RU](https://www.viva64.com/ru/b/0606/)).
