---
title: oitofelix - Article&#58; Savannah CVS to Git migration
description: >
  The Savannah CVS to Git migration article guides the reader through
  the necessary steps in migrating a Savannah hosted project that uses
  CVS as its version control system to the more modern and powerful
  Git, preserving all the project's history and improving the user
  naming in commit records.
tags: article, free software, Savannah, CVS, Git
license: CC BY-SA 4.0
layout: oitofelix-homepage
---
<div id="markdown" markdown="1">
## Article: Savannah CVS to Git migration

The _Savannah CVS to Git migration_ article guides the reader through
the necessary steps in migrating a Savannah hosted project that uses
CVS as its VCS (Version Control System) to the more modern and
powerful Git, preserving all the project's history and improving the
user naming in commit records.

- [Savannah](http://savannah.gnu.org/) is the
[GNU project](http://www.gnu.org/)'s software forge, also available
for the free software community at large.
- [CVS](http://cvs.nongnu.org/) is the _Concurrent Version System_, a
traditional and popular centralized VCS.
- [Git](http://git-scm.com/) is _the stupid content tracker_, a modern
and widely adopted distributed VCS.

This article presents two alternative methods, each one based on a
particular migration tool, that can be used to accomplish CVS to Git
repository conversion: `cvs2git` and `cvs-fast-export`.  You can
experiment with both and choose which one suits you best.

- [cvs2git](http://www.mcs.anl.gov/~jacob/cvs2svn/cvs2git.html) is
  part of the larger [cvs2svn](http://cvs2svn.tigris.org/) package.
- [cvs-fast-export](http://www.catb.org/~esr/cvs-fast-export/) is a
  tool authored by Keith Packard and currently maintained by Eric
  S. Raymond.

A handful of command-line tools are used in the procedures described
in this article.  They must be properly installed in your computer,
and are likely to be available from your GNU/Linux distribution's
package repositories.  If any happen not to be, you'll have to fetch
its source code in order to build and install from there.  That should
be straightforward, though.  In addition to `cvs2git`,
`cvs-fast-export` and `git`, we'll use `rsync`.

- [rsync](http://rsync.samba.org/) is a tool designed for fast
  incremental remote files transfer and synchronization.

The commands that you need to type are preceded with a `$` sign.  The
command's output is shown in the lines immediately following it.  That
output resembles the one you would obtain by running the same command
adapted to your circumstances, but are likely different.  The
command-line and output pair are meant to be thought as a screenshot
of a terminal window, but for brevity's sake we'll omit repetitive
output by using the `[...]` ellipsis character sequence.

The original motivation for writing this article arose when I decided
to migrate [GNU ccd2cue](http://www.gnu.org/software/ccd2cue/), one
package I've authored and maintained for the GNU project, from its
original CVS-based code repository to a Git-based one, before I
started working on the package's new release (version 0.4 at the
time).  Therefore, for the sake of simplicity and comprehension, in
this article we'll assume we are working to solve that particular
problem case and thus the following meta-information holds about the
project in Savannah:

- __User name:__ oitofelix
- __Project name:__ ccd2cue
- __Domain:__ gnu.org

Needless to say, you'll have to adapt this information to the case at
hand; for that end you can consider those as meta-variables, if you
like.  For instance, if you are not a GNU maintainer your project is
probably hosted at the [non-GNU Savannah](http://savannah.nongnu.org/)
and therefore the domain must be regarded as `nongnu.org`.  Other
modifications that may be required (and I'm aware of) are explicitly
noted as such in their respective context.  However, unforeseen
circumstances might arise from differences in repository structure,
run-time environment, project requirements, server-side modifications,
among other factors.  Therefore, be warned that your mileage may vary.
Furthermore, this article is distributed in the hope that it will be
useful, but __without any warranty__; without even the implied
warranty of __merchantability__ or __fitness for a particular
purpose__.

I'd like to thank __Assaf Gordon__, the very helpful Savannah hacker
whose expertise with `cvs-fast-export` is the basis for this article
on that matter.


### Obtaining CVS repository from Savannah

Firstly, we need to obtain from Savannah a local copy of the entire
project's CVS repository.  We'll use `rsync` in order to do that:

<pre>
$ rsync -av rsync://cvs.savannah.gnu.org/sources/ccd2cue .
receiving incremental file list
ccd2cue/
ccd2cue/CVSROOT/
ccd2cue/CVSROOT/checkoutlist
ccd2cue/CVSROOT/commitinfo
[...]
ccd2cue/ccd2cue/src/Attic/error.c,v
ccd2cue/ccd2cue/src/Attic/error.h,v

sent 2,685 bytes  received 1,907,505 bytes  103,253.51 bytes/sec
total size is 1,897,875  speedup is 0.99
</pre>

If everything went well the local directory `ccd2cue` should contain
the CVS repository.

<pre>
$ ls ccd2cue/
ccd2cue  CVSROOT
</pre>





### Enabling Git repository at Savannah and cloning it

It's necessary to enable the Git repository at Savannah and clone it
so we can import the converted repository and push it back.  In order
to do that, we have to go to
[Savannah project's feature selection page](https://savannah.gnu.org/project/admin/editgroupfeatures.php?group=ccd2cue)
and click to check the "Git" option like in the picture below:

![Savannah project's features selection page](savannah-project-features.png)

Wait about 30 minutes until the repository creation job is tackled by
the server and then clone the repository into the `ccd2cue.git`
directory --- not `ccd2cue` since it is occupied by the copy of
ccd2cue's CVS repository made in the previous step.

<pre>
$ git clone oitofelix@git.sv.gnu.org:/srv/git/ccd2cue.git ccd2cue.git
Cloning into 'ccd2cue.git'...
warning: You appear to have cloned an empty repository.
Checking connectivity... done.
</pre>

If the server hasn't completed the Git repository creation we'll see
instead an error message.

<pre>
$ git clone oitofelix@git.sv.gnu.org:/srv/git/ccd2cue.git ccd2cue.git
Cloning into 'ccd2cue.git'...
fatal: '/srv/git/ccd2cue.git' does not appear to be a git repository
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
</pre>

We have to keep trying from time to time until we succeed.





### Using `cvs2git` to convert the repository

Now it's time to do the actual conversion to a Git repository.  You
can use the method described in this section or go to
[cvs-fast-export section](#using-cvs-fast-export-to-convert-the-repository)
for an alternative method.

The `cvs2git` conversion process is driven by the so called "options
file".  That file is a regular Python program that can be used to
fine-tune the conversion.  The easiest and practical way to get
started in writing it is to modify the extensively commented options
file distributed along the `cvs2svn` package.  In my computer this
file is located at
`/usr/share/doc/cvs2svn/examples/cvs2git-example.options.gz`.


#### Setting up `cvs2git` options file

To produce a working options file, which can give us good results for
this conversion, we just need to make half dozen changes or so to the
vanilla options file.  Below are the necessary changes in unified diff
format grouped by their intention.


__Define CVS repository directory and unset temporary directory__

The CVS local repository directory is the `ccd2cue` directory fetched
from Savannah at the last step.

<pre>
@@ -560,7 +550,7 @@
     # The filesystem path to the part of the CVS repository (*not* a
     # CVS working copy) that should be converted.  This may be a
     # subdirectory (i.e., a module) within a larger CVS repository.
-    r'test-data/main-cvsrepos',
+    r'ccd2cue',
 
     # A list of symbol transformations that can be used to rename
     # symbols in this project.
</pre>

The temporary directory is where `cvs2git` outputs the resulting
files.  We want them to be placed in the current working directory,
thus we unset it.

<pre>
@@ -122,8 +122,6 @@
 #logger.log_level = logger.DEBUG
 
 
-# The directory to use for temporary files:
-ctx.tmpdir = r'cvs2svn-tmp'
 
 # During FilterSymbolsPass, cvs2git records the contents of file
 # revisions into a "blob" file in git-fast-import format.  The
</pre>


__Define `blob` and `dump` output file names__

The whole conversion process outputs two files, which must be fed to
`git fast-import`.  The `blob` file comprises the revision contents.

<pre>
@@ -135,7 +133,7 @@
 ctx.revision_collector = GitRevisionCollector(
     # The file in which to write the git-fast-import stream that
     # contains the file revision contents:
-    'cvs2svn-tmp/git-blob.dat',
+    'blob',
 
     # The following option specifies how the revision contents of the
     # RCS files should be read.
</pre>

The `dump` file comprises the change-sets and branch/tag information.

<pre>
@@ -528,7 +518,7 @@
 ctx.output_option = GitOutputOption(
     # The file in which to write the git-fast-import stream that
     # contains the changesets and branch/tag information:
-    os.path.join(ctx.tmpdir, 'git-dump.dat'),
+    'dump',
 
     # The blobs will be written via the revision recorder, so in
     # OutputPass we only have to emit references to the blob marks:
</pre>


__Set symbol transformation rules__

When moving to Git it's a good practice to tag the `HEAD` of the
repository with something like `cvs-repository-moved-to-git`, so
people reaching it can see that the repository is not being updated
anymore.  The change below prevents `cvs2git` from generating the same
tag in the Git repository.

<pre>
@@ -575,6 +565,7 @@
         # branches correctly.  The argument is a Python-style regular
         # expression that has to match the *whole* CVS symbol name:
         #IgnoreSymbolTransform(r'nightly-build-tag-.*')
+        IgnoreSymbolTransform(r'cvs-repository-moved-to-git'),
 
         # RegexpSymbolTransforms transform symbols textually using a
         # regular expression.  The first argument is a Python regular
</pre>

Tag names in CVS are quite restrictive while in Git they are a lot
more permissive, thus we'll transform the old release tags `rel_0-1`,
`rel_0-2`,... into the more appropriate `0.1`, `0.2` and so on.

<pre>
@@ -591,6 +582,7 @@
         #                      r'release-\1.\2'),
         #RegexpSymbolTransform(r'release-(\d+)_(\d+)_(\d+)',
         #                      r'release-\1.\2.\3'),
+        RegexpSymbolTransform(r'rel_(\d+)-(\d+)', r'\1.\2'),
 
         # Simple 1:1 character replacements can also be done.  The
         # following transform, which converts backslashes into forward
</pre>


__Map CVS users to full names and emails__

CVS uses Unix user names in commit records, while Git allows full name
plus an email address.  It's helpful to make use of that additional
feature thus we'll map one into another.  For that end we need to
first obtain a list of all the users that have commited to the CVS
repository.

<pre>
$ sed 's/^[^|]*|\([^|]*\)|.*$/\1/' ccd2cue/CVSROOT/history | uniq
oitofelix
</pre>

With this list in hands we can create the mapping in the options file.

<pre>
@@ -512,15 +510,7 @@
 # (name, email).  Please substitute your own project's usernames here
 # to use with the author_transforms option of GitOutputOption below.
 author_transforms={
-    'jrandom' : ('J. Random', 'jrandom@example.com'),
-    'mhagger' : 'Michael Haggerty <mhagger@alum.mit.edu>',
-    'brane' : (u'Branko Čibej', 'brane@xbc.nu'),
-    'ringstrom' : 'Tobias Ringström <tobias@ringstrom.mine.nu>',
-    'dionisos' : (u'Erik Hülsmann', 'e.huelsmann@gmx.net'),
-
-    # This one will be used for commits for which CVS doesn't record
-    # the original author, as explained above.
-    'cvs2svn' : 'cvs2svn <admin@example.com>',
+    'oitofelix' : 'Bruno Félix Rezende Ribeiro <oitofelix@gnu.org>',
     }
 
 # This is the main option that causes cvs2svn to output to a
</pre>


#### Running `cvs2git`

The `cvs2git` options file has all the necessary settings to guide the
conversion, therefore in `cvs2git` invocation no additional arguments
are required besides `--options`, which we use to specify the file
created at the previous step.  We assume it's named `options` and has
been placed in the current working directory.

<pre>
$ cvs2git --options=options
----- pass 1 (CollectRevsPass) -----
Examining all CVS ',v' files...
ccd2cue/ccd2cue/.cvsignore,v
[...]
cvs2svn Statistics:
------------------
Total CVS Files:               116
Total CVS Revisions:           444
Total CVS Branches:              0
Total CVS Tags:                140
Total Unique Tags:               5
Total Unique Branches:           0
CVS Repos Size in KB:         1816
Total SVN Commits:             154
First Revision Date:    Fri Mar 18 09:45:50 2011
Last Revision Date:     Tue Feb 11 15:13:46 2014
------------------
Timings (seconds):
------------------
0.936   pass1    CollectRevsPass
0.072   pass2    CleanMetadataPass
0.015   pass3    CollateSymbolsPass
5.734   pass4    FilterSymbolsPass
0.056   pass5    SortRevisionsPass
0.028   pass6    SortSymbolsPass
0.278   pass7    InitializeChangesetsPass
0.202   pass8    BreakRevisionChangesetCyclesPass
0.200   pass9    RevisionTopologicalSortPass
0.100   pass10   BreakSymbolChangesetCyclesPass
0.202   pass11   BreakAllChangesetCyclesPass
0.182   pass12   TopologicalSortPass
0.421   pass13   CreateRevsPass
0.022   pass14   SortSymbolOpeningsClosingsPass
0.019   pass15   IndexSymbolsPass
0.314   pass16   OutputPass
8.781   total
</pre>

Two files should have been generated in the current directory: `blob`
and `dump`.  To create an auto-sufficient _Git fast-import file_ we
need to concatenate both.

<pre>
cat blob dump > fast-import-file
</pre>

The `blob` and `dump` files are no longer necessary and can be wiped
out at will.




### Using `cvs-fast-export` to convert the repository

If you have chosen the `cvs2git` method you can skip to the
[next section](#importing-the-repository-and-pushing-it-back-to-savannah),
otherwise continue reading on.

CVS uses Unix user names in commit records, while Git allows full name
plus an email address.  It's helpful to make use of that additional
feature thus we'll map one into another.  For that end we need to
first obtain a list of all the users that have commited to the CVS
repository.

<pre>
$ find ccd2cue -type f | cvs-fast-export -a
oitofelix
</pre>

With this list in hands we can create a mapping file to guide
`cvs-fast-export` on how to transform user identities.  We can use any
editor of our choice or, alternatively, if the user name list is
small, we can generate that file straight from the command-line.

<pre>
$ cat > author-map << EOF
> oitofelix=Bruno Félix Rezende Ribeiro <oitofelix@gnu.org>
> EOF
</pre>

The last step in this method is to invoke `cvs-fast-export` in order
to produce the _Git fast-import file_.

<pre>
$ find ccd2cue -type f | cvs-fast-export -A author-map > fast-import-file
</pre>





### Importing the repository and pushing it back to Savannah

Finally, importing the converted repository is as simple as feeding
`git fast-import` with the file generated by `cvs2git` or
`cvs-fast-export` conversion tools.

<pre>
$ cd ccd2cue.git && git fast-import < ../fast-import-file
git-fast-import statistics:
---------------------------------------------------------------------
Alloc'd objects:       5000
Total objects:          940 (        22 duplicates                  )
      blobs  :          373 (        21 duplicates        310 deltas of        350 attempts)
      trees  :          419 (         1 duplicates        233 deltas of        399 attempts)
      commits:          148 (         0 duplicates          0 deltas of          0 attempts)
      tags   :            0 (         0 duplicates          0 deltas of          0 attempts)
Total branches:           4 (         1 loads     )
      marks:     1073741824 (       542 unique    )
      atoms:             96
Memory total:          2294 KiB
       pools:          2098 KiB
     objects:           195 KiB
---------------------------------------------------------------------
pack_report: getpagesize()            =       4096
pack_report: core.packedGitWindowSize =   33554432
pack_report: core.packedGitLimit      =  268435456
pack_report: pack_used_ctr            =         11
pack_report: pack_mmap_calls          =          4
pack_report: pack_open_windows        =          1 /          1
pack_report: pack_mapped              =     623459 /     623459
---------------------------------------------------------------------
</pre>

If you have `gitk` installed you can run `gitk --all` to inspect the
repository's sanity and health.  Otherwise you can use just a simple
`git log` if that's enough.  You may also find useful to re-compact
the repository and discard any garbage.

<pre>
$ git gc --prune=now
Counting objects: 940, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (238/238), done.
Writing objects: 100% (940/940), done.
Total 940 (delta 543), reused 940 (delta 543)
</pre>

Our working tree is empty; let's populate it.

<pre>
$ git checkout master
Already on 'master'
Your branch is based on 'origin/master', but the upstream is gone.
  (use "git branch --unset-upstream" to fixup)
</pre>

Finally, we have to push the whole repository back to the remote.

<pre>
$ git push --all && git push --tags
Counting objects: 792, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (238/238), done.
Writing objects: 100% (792/792), 602.45 KiB | 0 bytes/s, done.
Total 792 (delta 543), reused 792 (delta 543)
To oitofelix@git.sv.gnu.org:/srv/git/ccd2cue.git
 * [new branch]      master -> master
 * [new tag]         rel_0-1 -> rel_0-1
 * [new tag]         rel_0-2 -> rel_0-2
 * [new tag]         rel_0-3 -> rel_0-3
</pre>

Now, it's all done.  __Happy hacking!__


</div>
