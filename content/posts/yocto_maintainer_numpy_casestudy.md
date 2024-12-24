+++
title = "Maintaining the Yocto Project: Case Study: python3-numpy"
date = 2021-01-26
+++

## Introduction

[Yocto Project](https://www.yoctoproject.org/) maintenance frequently
involves upgrading many recipes in a layer to a more recent upstream
version. The benefits of doing so include pulling in bugfixes, new
features and tests, retaining compatibility with other packages, and so
on. Manually, this might be done with six or seven main steps as
follows:

1. Determine the appropriate version (usually a stable release) to
   upgrade to;
2. Use `git mv` to change the recipe file's name to reflect the new
   version;
3. Change recipe variables such as `SRCREV`, `SRC_URI`,
   `LIC_FILES_CHKSUM`, and so on (as required) to match;
4. Setup a build area with `source oe-init-build-env` and confirm that
   the recipe builds with no new warnings or errors;
5. Build the recipe to be sure that there are no errors and rule out
   unexpected license or QA changes;
6. If it has ptests, build `core-image-ptest-<recipe>`, run the image,
   and check that there are no regressions when running `ptest-runner
   <recipe>`;
7. Compile all of the changes made into a patch with a description of
   what you've done and submit it for review.

However, what happens when steps 4, 5, and/or 6 don't go smoothly? In
this post we'll walk through an already-completed upgrade for the
`python3-numpy` recipe, which will be illustrative as to why and what
might need to be done. Every recipe and associated project is different,
though, so what you see here might not work elsewhere.

## Some Background Context

I've been maintainer for the `python3-numpy` recipe in the openembedded-core
layer [since August
2023](https://git.openembedded.org/openembedded-core/commit/?id=3a7021f5029ad30f5cf9adf02c91029e63ef0ef8).
It turns out that this is one of the more complicated Python recipes to
maintain, given that much of the module is compiled, rather than simply
being redistributed Python code. Pretty much every new upstream release
results in one or more of the following upgrade blockers occurring for
the recipe:

1. Carried patches (backports, embedded-specific, etc.) no longer
apply, because: 
    1. the target files have changed too much for the
appropriate modifications to be completed in do_patch;
    2. the changes are already present (i.e. the patch was a backport and its
changes are included in the release you're upgrading to);
2. Architecture-specific build issues, which are often already
identified and patched, but not included in the latest available numpy
relase;
3. New reproducibility issues are identified;
4. Breakages now occur other recipes that depend on it.

We'll see a few of these below, as we walk through the upgrade from
numpy version `1.26.4` to `2.1.3`.

## Setup

Since this upgrade has been merged into master for a while, let's find
it using `git log --oneline --grep="numpy"`:

```
e0e5dd1e434 python3-numpy: upgrade 2.2.1 -> 2.2.2
991cbf2336e python3-numpy: upgrade 2.1.3 -> 2.2.1
e16e725fb51 Revert "python3-numpy: upgrade 2.1.3 -> 2.2.0"
48b0c58b34e python3-numpy: upgrade 2.1.3 -> 2.2.0
0b625064112 python3-numpy: inherit pkgconfig
ea8594d4a0b python3-numpy: upgrade 1.26.4 -> 2.1.3
0650c1194e5 python3-hypothesis: upgrade 6.111.2 -> 6.112.1
...
```

We can see the commit ID is `ea8594d4a0b`. Since we want to walk through
the process ourselves, we need to checkout the commit before it:

`git checkout ea8594d4a0b^`

And now the output of `git log --oneline` looks like:

```
edfab1a3ceb (HEAD) python3-meson-python: upgrade 0.16.0 -> 0.17.1
c584cfaa3fe meson: don't look on the host for GTest when cross-compiling
d3576eab8ec linux: Modify kernel configuration to fix runqlat issue
9fe2d717541 migration-guides: document ZSTD_COMPRESSION_LEVEL change
9620944507f ref-manual: document ZSTD_COMPRESSION_LEVEL
b95959a56bc ref-manual: merge two separate descriptions of RECIPE_UPGRADE_EXTRA_TASKS
a11bc1d3209 migration-guides: add release notes for 5.0.4
...
```

## The Upgrade

Let's start by modifying the recipe file and starting bitbake on the
recipe. We know this won't work since the sha256sum in the recipe hasn't
been changed to match, but it'll tell us what bitbake sees when it tries
to download the new package version, and we can check it to make sure
it's correct:

```
tgamblin@megalith ~/workspace/yocto/poky ((HEAD detached at edfab1a3ceb))$ git mv meta/recipes-devtools/python/python3-numpy_1.26.4.bb meta/recipes-devtools/python/python3-numpy_2.1.3.bb
tgamblin@megalith ~/workspace/yocto/poky ((HEAD detached at edfab1a3ceb))$ git status
HEAD detached at edfab1a3ceb
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        renamed:    meta/recipes-devtools/python/python3-numpy_1.26.4.bb -> meta/recipes-devtools/python/python3-numpy_2.1.3.bb

tgamblin@megalith ~/workspace/yocto/poky ((HEAD detached at edfab1a3ceb))$ source oe-init-build-env build/
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$ bitbake python3-numpy
Loading cache: 100% |                                                                                                                                                                                                                         | ETA:  --:--:--
Loaded 0 entries from dependency cache.
Parsing recipes:  98% |#####################################################################################################################################################################################################################   | ETA:  0:00:00

...
```

And the expected error:

```
WARNING: python3-numpy-2.1.3-r0 do_fetch: Renaming /home/tgamblin/workspace/yocto/poky/build/downloads/numpy-2.1.3.tar.gz to /home/tgamblin/workspace/yocto/poky/build/downloads/numpy-2.1.3.tar.gz_bad-checksum_aa08e04e08aaf974d4458def539dece0d28146d866a31
WARNING: python3-numpy-2.1.3-r0 do_fetch: Checksum failure encountered with download of https://github.com/numpy/numpy/releases/download/v2.1.3/numpy-2.1.3.tar.gz - will attempt other sources if available
WARNING: python3-numpy-2.1.3-r0 do_fetch: Checksum mismatch for local file /home/tgamblin/workspace/yocto/poky/build/downloads/numpy-2.1.3.tar.gz
Cleaning and trying again.
WARNING: python3-numpy-2.1.3-r0 do_fetch: Renaming /home/tgamblin/workspace/yocto/poky/build/downloads/numpy-2.1.3.tar.gz to /home/tgamblin/workspace/yocto/poky/build/downloads/numpy-2.1.3.tar.gz_bad-checksum_aa08e04e08aaf974d4458def539dece0d28146d866a31
ERROR: python3-numpy-2.1.3-r0 do_fetch: Checksum failure fetching https://github.com/numpy/numpy/releases/download/v2.1.3/numpy-2.1.3.tar.gz
ERROR: python3-numpy-2.1.3-r0 do_fetch: Bitbake Fetcher Error: ChecksumError('Checksum mismatch!\nFile: \'/home/tgamblin/workspace/yocto/poky/build/downloads/numpy-2.1.3.tar.gz\' has sha256 checksum \'aa08e04e08aaf974d4458def539dece0d28146d866a39da56395)
ERROR: Logfile of failure stored in: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_fetch.3240682
ERROR: Task (/home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_fetch) failed with exit code '1'
NOTE: Tasks Summary: Attempted 1358 tasks of which 999 didn't need to be rerun and 1 failed.

Summary: 1 task failed:
  /home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_fetch
    log: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_fetch.3240682
Summary: There were 5 WARNING messages.
Summary: There were 2 ERROR messages, returning a non-zero exit code.
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$
```

So we modify the `SRC_URI` line in the recipe file (making sure we get
the full hash, as some terminals like mine may cut the output):

```
~ SRC_URI[sha256sum] = "aa08e04e08aaf974d4458def539dece0d28146d866a39da5639596f4921fd761"
```

### Patch Tweaks

Now we try bitbake again:

```
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$ bitbake python3-numpy
...

NOTE: Executing Tasks
ERROR: python3-numpy-2.1.3-r0 do_patch: Applying patch '0001-numpy-core-Define-RISCV-32-support.patch' on target directory '/home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/numpy-2.1.3'
CmdError('quilt --quiltrc /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/recipe-sysroot-native/etc/quiltrc push', 0, "stdout: Applying patch 0001-numpy-core-Define-RISCV-32-support.patch
can't find file to patch at input line 19
Perhaps you used the wrong -p or --strip option?
The text leading up to this was:
--------------------------
|From eb6d6579150bf4684603ce377c51e90ad3bb8109 Mon Sep 17 00:00:00 2001
|From: Khem Raj <raj.khem@gmail.com>
|Date: Sun, 15 Nov 2020 15:32:39 -0800
|Subject: [PATCH] numpy/core: Define RISCV-32 support
|
|Helps compile on riscv32
|
|Upstream-Status: Submitted [https://github.com/numpy/numpy/pull/17780]
|Signed-off-by: Khem Raj <raj.khem@gmail.com>
|---
| numpy/core/include/numpy/npy_cpu.h    | 3 +++
| numpy/core/include/numpy/npy_endian.h | 1 +
| 2 files changed, 4 insertions(+)
|
|diff --git a/numpy/core/include/numpy/npy_cpu.h b/numpy/core/include/numpy/npy_cpu.h
|index 78d229e..04be511 100644
|--- a/numpy/core/include/numpy/npy_cpu.h
|+++ b/numpy/core/include/numpy/npy_cpu.h
--------------------------
No file to patch.  Skipping patch.
2 out of 2 hunks ignored
can't find file to patch at input line 40
Perhaps you used the wrong -p or --strip option?
The text leading up to this was:
--------------------------
|diff --git a/numpy/core/include/numpy/npy_endian.h b/numpy/core/include/numpy/npy_endian.h
|index 5e58a7f..0926212 100644
|--- a/numpy/core/include/numpy/npy_endian.h
|+++ b/numpy/core/include/numpy/npy_endian.h
--------------------------
No file to patch.  Skipping patch.
1 out of 1 hunk ignored
Patch 0001-numpy-core-Define-RISCV-32-support.patch does not apply (enforce with -f)

stderr: ")
ERROR: Logfile of failure stored in: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_patch.3277955
ERROR: Task (/home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_patch) failed with exit code '1'
NOTE: Tasks Summary: Attempted 1394 tasks of which 1363 didn't need to be rerun and 1 failed.

Summary: 1 task failed:
  /home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_patch
    log: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_patch.3277955
Summary: There was 1 ERROR message, returning a non-zero exit code.
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$
```

So one of the patches we carry fails to apply now. A look at the
upstream numpy repo shows that the PR mentioned in its commit message
has been merged, but isn't in this release (at this point it's showing
up in v2.2.x and later tags), seen
[here](https://github.com/numpy/numpy/pull/17780/commits/0e2b652a0eff85798584116c905a2d6ad8f25d5f).

Usually when something like this occurs, you can fix it by doing the
following:

1. Cloning a copy of the upstream repo
2. Checking out the tag for your target version (in this case, `v2.1.3`)
3. Doing `git am /path/to/the/yocto/recipe/patch/file.patch`
4. Resolving conflicts manually and/or with the help of a tool such as
[wiggle](https://github.com/neilbrown/wiggle)

Let's try that:

```
tgamblin@megalith ~/workspace/git/pythonsrc/numpy (main)$ git checkout v2.1.3
Note: switching to 'v2.1.3'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

HEAD is now at 98464cc0cb Merge pull request #27690 from charris/prepare-2.1.3

tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ git am ~/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy/0001-numpy-core-Define-RISCV-32-support.patch
Applying: numpy/core: Define RISCV-32 support
error: numpy/core/include/numpy/npy_cpu.h: does not exist in index
error: numpy/core/include/numpy/npy_endian.h: does not exist in index
Patch failed at 0001 numpy/core: Define RISCV-32 support
hint: Use 'git am --show-current-patch=diff' to see the failed patch
When you have resolved this problem, run "git am --continue".
If you prefer to skip this patch, run "git am --skip" instead.
To restore the original branch and stop patching, run "git am --abort".
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ wiggle -r -p ~/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy/0001-numpy-core-Define-RISCV-32-support.patch
wiggle: Cannot find files to patch: please specify --strip
wiggle: aborting
```

Interestingly, it seems that the files can't be found at all. Seems odd,
right? A quick glance at the `numpy` subdirectory inside that project
repo shows us why (`core` has been renamed to `_core`):

```
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached from v2.1.3))$ ls numpy/
_array_api_info.py   compat            _core          _distributor_init.py  dtypes.pyi             f2py                    __init__.pxd  linalg     meson.build       _pytesttester.pyi  strings  typing
_array_api_info.pyi  __config__.py.in  core           distutils             exceptions.py          fft                     __init__.py   ma         polynomial        py.typed           testing  _utils
_build_utils         _configtool.py    ctypeslib.py   doc                   exceptions.pyi         _globals.py             __init__.pyi  matlib.py  _pyinstaller      random             tests    version.pyi
char                 conftest.py       ctypeslib.pyi  dtypes.py             _expired_attrs_2_0.py  __init__.cython-30.pxd  lib           matrixlib  _pytesttester.py  rec                _typing
```

So we can modify the paths inside the patch file to match. Then it
applies:

```
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ vim ~/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy/0001-numpy-core-Define-RISCV-32-support.patch
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ wiggle -r -p ~/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy/0001-numpy-core-Define-RISCV-32-support.patch
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ git status
HEAD detached at v2.1.3
You are in the middle of an am session.
  (fix conflicts and then run "git am --continue")
  (use "git am --skip" to skip this patch)
  (use "git am --abort" to restore the original branch)

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   numpy/_core/include/numpy/npy_cpu.h
        modified:   numpy/_core/include/numpy/npy_endian.h

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        numpy/_core/include/numpy/npy_cpu.h.porig
        numpy/_core/include/numpy/npy_endian.h.porig
        venv/

no changes added to commit (use "git add" and/or "git commit -a")
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ git add numpy/_core/include/numpy/npy_cpu.h numpy/_core/include/numpy/npy_endian.h
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ git am --continue
Applying: numpy/core: Define RISCV-32 support
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached from v2.1.3))$ git format-patch -1
0001-numpy-core-Define-RISCV-32-support.patch
```

At this point, update the carried version of the patch:

```
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached from v2.1.3))$ cp 0001-numpy-core-Define-RISCV-32-support.patch ~/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy/0001-numpy-core-Define-RISCV-32-support.patch
```

Now it should apply. In a real upgrade scenario, you'd also want to
modify that patch's commit message to say what was done, e.g. at least
to say it was reworked to apply on version v2.1.3.

### License Update!

Let's try building again:

```
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$ bitbake python3-numpy
Loading cache: 100% |##########################################################################################################################################################################################################################| Time: 0:00:00
Loaded 1969 entries from dependency cache.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.9.1"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "x86_64-poky-linux"
MACHINE              = "qemux86-64"
DISTRO               = "poky"
DISTRO_VERSION       = "5.1"
TUNE_FEATURES        = "m64 core2"
TARGET_FPU           = ""
meta
meta-poky
meta-yocto-bsp
meta-selftest        = "HEAD:edfab1a3cebc454e4d63caae0134e02d169260c6"

Sstate summary: Wanted 258 Local 11 Mirrors 0 Missed 247 Current 464 (4% match, 65% complete)############################################################################################################################                      | ETA:  0:00:00
Initialising tasks: 100% |#####################################################################################################################################################################################################################| Time: 0:00:00
NOTE: Executing Tasks
ERROR: python3-numpy-2.1.3-r0 do_populate_lic: QA Issue: python3-numpy: The LIC_FILES_CHKSUM does not match for file://LICENSE.txt;md5=a752eb20459cf74a9d84ee4825e8317c
python3-numpy: The new md5 checksum is 1de863c37a83e71b1e97b64d036ea78b
python3-numpy: Here is the selected license text:
vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
Copyright (c) 2005-2024, NumPy Developers.
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are
met:

    * Redistributions of source code must retain the above copyright
       notice, this list of conditions and the following disclaimer.

...
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
python3-numpy: Check if the license information has changed in /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/numpy-2.1.3/LICENSE.txt to verify that the LICENSE value "BSD-3-Clause & BSD-2-Clause & PSF-2.0 & A]
ERROR: python3-numpy-2.1.3-r0 do_populate_lic: Fatal QA errors were found, failing task.
ERROR: Logfile of failure stored in: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_populate_lic.3304105
ERROR: Task (/home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_populate_lic) failed with exit code '1'
NOTE: Tasks Summary: Attempted 1408 tasks of which 1392 didn't need to be rerun and 1 failed.

Summary: 1 task failed:
  /home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_populate_lic
    log: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_populate_lic.3304105
Summary: There were 2 ERROR messages, returning a non-zero exit code.
```

Right away we got another failure. Let's look at the diff between
`v1.26.4` and `v2.1.3` in the numpy repo to see what happened to the
`LICENSE.txt` file:

```
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached from v2.1.3))$ git diff v1.26.4..v2.1.3 LICENSE.txt
diff --git a/LICENSE.txt b/LICENSE.txt
index 014d51c98d..6ccec6824b 100644
--- a/LICENSE.txt
+++ b/LICENSE.txt
@@ -1,4 +1,4 @@
-Copyright (c) 2005-2023, NumPy Developers.
+Copyright (c) 2005-2024, NumPy Developers.
 All rights reserved.

 Redistribution and use in source and binary forms, with or without
 ```

It's only the copyright year that changed, but we'll need to go and edit
the numpy recipe file again to update `LIC_FILES_CHKSUM`:

```
~ LIC_FILES_CHKSUM = "file://LICENSE.txt;md5=1de863c37a83e71b1e97b64d036ea78b"
```

This change should also be indicated in the upgrade patch's commit
message, with a line like:

```
License-Update: Change copyright year to 2024
```

### Build Backend Change

Let's invoke bitbake yet again:

```
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$ bitbake python3-numpy
Loading cache: 100% |##########################################################################################################################################################################################################################| Time: 0:00:00
Loaded 1969 entries from dependency cache.
Parsing recipes: 100% |########################################################################################################################################################################################################################| Time: 0:00:00
Parsing of 993 .bb files complete (992 cached, 1 parsed). 1969 targets, 56 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.9.1"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "x86_64-poky-linux"
MACHINE              = "qemux86-64"
DISTRO               = "poky"
DISTRO_VERSION       = "5.1"
TUNE_FEATURES        = "m64 core2"
TARGET_FPU           = ""
meta
meta-poky
meta-yocto-bsp
meta-selftest        = "HEAD:edfab1a3cebc454e4d63caae0134e02d169260c6"

Sstate summary: Wanted 256 Local 11 Mirrors 0 Missed 245 Current 466 (4% match, 66% complete)############################################################################################################################                      | ETA:  0:00:00
Removing 1 stale sstate objects for arch core2-64: 100% |######################################################################################################################################################################################| Time: 0:00:00
NOTE: Executing Tasks
...
```

This time it takes a lot longer before we get any sort of feedback, but
when we do (surprise!), it's another error:

```
ERROR: python3-numpy-2.1.3-r0 do_compile: 'python3 setup.py bdist_wheel ' execution failed.
ERROR: python3-numpy-2.1.3-r0 do_compile: Execution of '/home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/run.do_compile.4016285' failed with exit code 1
ERROR: Logfile of failure stored in: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_compile.4016285
Log data follows:
| DEBUG: Executing shell function do_compile
| /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/recipe-sysroot-native/usr/bin/python3-native/python3: can't open file '/home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.y
| ERROR: 'python3 setup.py bdist_wheel ' execution failed.
| WARNING: exit code 1 from a shell command.
ERROR: Task (/home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_compile) failed with exit code '1'
```

It appears that when `do_compile` tries to go and make use of numpy's
`setup.py` file, it can't find it. Looking at the top-level repo
contents, it's indeed missing, but what you may notice is that there are
files present for the `meson` build backend:

```
tgamblin@megalith ~/workspace/git/pythonsrc/numpy ((HEAD detached at v2.1.3))$ ls
0001-numpy-core-Define-RISCV-32-support.patch  azure-steps-windows.yml  branding                CITATION.bib  environment.yml  LICENSES_bundled.txt  meson.build  meson_options.txt  pavement.py     pytest.ini  requirements  tools           venv
azure-pipelines.yml                            benchmarks               building_with_meson.md  doc           INSTALL.rst      LICENSE.txt           meson_cpu    numpy              pyproject.toml  README.md   THANKS.txt    vendored-meson
```

So let's try updating the recipe file to use that now. For Python
recipes, you can do this by replacing `setuptools3` in the `inherit`
line with `python_mesonpy`:

```
~ inherit ptest python_mesonpy github-releases cython
```

### Round and Round

After another bitbake invocation:

```
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$ bitbake python3-numpy                                                                                                                                                        
Loading cache: 100% |##########################################################################################################################################################################################################################| Time: 0:00:00
Loaded 1969 entries from dependency cache.                                                                                                                                                                                                                    
Parsing recipes: 100% |########################################################################################################################################################################################################################| Time: 0:00:00
Parsing of 993 .bb files complete (992 cached, 1 parsed). 1969 targets, 56 skipped, 0 masked, 0 errors.                                                                                                                                                       
NOTE: Resolving any missing task queue dependencies                                                                                                                                                                                                           
                                                                                                                                                                                                                                                              
Build Configuration:                                                                                                                                                                                                                                          
BB_VERSION           = "2.9.1"                                                                                                                                                                                                                                
BUILD_SYS            = "x86_64-linux"                                                                                                                                                                                                                         
NATIVELSBSTRING      = "universal"                                                                                                                                                                                                                            
TARGET_SYS           = "x86_64-poky-linux"                                                                                                                                                                                                                    
MACHINE              = "qemux86-64"                                                                                                                                                                                                                           
DISTRO               = "poky"                                                                                                                                                                                                                                 
DISTRO_VERSION       = "5.1"                                                                                                                                                                                                                                  
TUNE_FEATURES        = "m64 core2"                                                                                                                                                                                                                            
TARGET_FPU           = ""                                                                                                                                                                                                                                     
meta                                                                                                                                                                                                                                                          
meta-poky                                                                                                                                                                                                                                                     
meta-yocto-bsp                                                                                                                                                                                                                                                
meta-selftest        = "HEAD:edfab1a3cebc454e4d63caae0134e02d169260c6"                                                                                                                                                                                        
                                                                                                                                                                                                                                                              
Sstate summary: Wanted 92 Local 18 Mirrors 0 Missed 74 Current 655 (19% match, 90% complete)#############################################################################################################################                      | ETA:  0:00:00
Removing 7 stale sstate objects for arch x86_64: 100% |########################################################################################################################################################################################| Time: 0:00:00
Removing 1 stale sstate objects for arch core2-64: 100% |######################################################################################################################################################################################| Time: 0:00:00
NOTE: Executing Tasks                                                                                                                                                                                                                                         
ERROR: python3-numpy-2.1.3-r0 do_compile: Execution of '/home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/run.do_compile.4143925' failed with exit code 1
ERROR: Logfile of failure stored in: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_compile.4143925                                                                                                   
Log data follows:                                                                                                                                                                                                                                             
| DEBUG: Executing shell function do_compile       
| * Getting build dependencies for wheel...                                                                                                                                                                                                                   
| * Building wheel...                                                                                                          
| + /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/recipe-sysroot-native/usr/bin/nativepython3 /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/numpy-2.1.3/vendored-mei
| The Meson build system             
| Version: 1.5.2                  
| Source dir: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/numpy-2.1.3
| Build dir: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/build
| Build type: cross build    
| Project name: NumPy       
| Project version: 2.1.3          
| C compiler for the host machine: x86_64-poky-linux-gcc -m64 -march=core2 -mtune=core2 -msse3 -mfpmath=sse -fstack-protector-strong -O2 -D_FORTIFY_SOURCE=2 -Wformat -Wformat-security -Werror=format-security --sysroot=/home/tgamblin/workspace/yocto/poky)
| C linker for the host machine: x86_64-poky-linux-gcc -m64 -march=core2 -mtune=core2 -msse3 -mfpmath=sse -fstack-protector-strong -O2 -D_FORTIFY_SOURCE=2 -Wformat -Wformat-security -Werror=format-security --sysroot=/home/tgamblin/workspace/yocto/poky/b1
| C++ compiler for the host machine: x86_64-poky-linux-g++ -m64 -march=core2 -mtune=core2 -msse3 -mfpmath=sse -fstack-protector-strong -O2 -D_FORTIFY_SOURCE=2 -Wformat -Wformat-security -Werror=format-security --sysroot=/home/tgamblin/workspace/yocto/po)
| C++ linker for the host machine: x86_64-poky-linux-g++ -m64 -march=core2 -mtune=core2 -msse3 -mfpmath=sse -fstack-protector-strong -O2 -D_FORTIFY_SOURCE=2 -Wformat -Wformat-security -Werror=format-security --sysroot=/home/tgamblin/workspace/yocto/poky1
| Cython compiler for the host machine: cython3 (cython 3.0.11)                                                                
| C compiler for the build machine: gcc (gcc 12.2.0 "gcc (Debian 12.2.0-14) 12.2.0")
| C linker for the build machine: gcc ld.bfd 2.40
| C++ compiler for the build machine: g++ (gcc 12.2.0 "g++ (Debian 12.2.0-14) 12.2.0")
| C++ linker for the build machine: g++ ld.bfd 2.40
| Cython compiler for the build machine: cython3 (cython 3.0.11)
| Build machine cpu family: x86_64
| Build machine cpu: x86_64
| Host machine cpu family: x86_64
| Host machine cpu: x86_64
| Target machine cpu family: x86_64
| Target machine cpu: x86_64
| Program python3 found: YES (/home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/recipe-sysroot-native/usr/bin/nativepython3)
| Did not find pkg-config by name 'pkg-config'
| Found pkg-config: NO
| Run-time dependency python found: NO (tried pkgconfig, pkgconfig and sysconfig)
| 
| ../numpy-2.1.3/meson.build:41:12: ERROR: Python dependency not found
| 
| A full log can be found at /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/build/meson-logs/meson-log.txt
| 
| ERROR Backend subprocess exited when trying to invoke build_wheel
| WARNING: exit code 1 from a shell command.
ERROR: Task (/home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_compile) failed with exit code '1'
NOTE: Tasks Summary: Attempted 2085 tasks of which 1949 didn't need to be rerun and 1 failed.

Summary: 1 task failed:
  /home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_compile
    log: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_compile.4143925
Summary: There was 1 ERROR message, returning a non-zero exit code.
```

Let's edit the `inherit` line again to look like:

```
~ inherit pkgconfig ptest python_mesonpy github-releases cython
```

and try again.

### Closer, Closer...

This time, when we try to build we see some QA warnings:

```
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$ bitbake python3-numpy
Loading cache: 100% |##########################################################################################################################################################################################################################| Time: 0:00:00
Loaded 1969 entries from dependency cache.
Parsing recipes: 100% |########################################################################################################################################################################################################################| Time: 0:00:00
Parsing of 993 .bb files complete (992 cached, 1 parsed). 1969 targets, 56 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.9.1"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "x86_64-poky-linux"
MACHINE              = "qemux86-64"
DISTRO               = "poky"
DISTRO_VERSION       = "5.1"
TUNE_FEATURES        = "m64 core2"
TARGET_FPU           = ""
meta
meta-poky
meta-yocto-bsp
meta-selftest        = "HEAD:edfab1a3cebc454e4d63caae0134e02d169260c6"

Sstate summary: Wanted 27 Local 10 Mirrors 0 Missed 17 Current 720 (37% match, 97% complete)#############################################################################################################################                      | ETA:  0:00:00
Initialising tasks: 100% |#####################################################################################################################################################################################################################| Time: 0:00:00
NOTE: Executing Tasks
ERROR: python3-numpy-2.1.3-r0 do_package_qa: QA Issue: File /usr/lib/python3.13/site-packages/numpy/__config__.py in package python3-numpy contains reference to TMPDIR [buildpaths]
ERROR: python3-numpy-2.1.3-r0 do_package_qa: QA Issue: non -staticdev package contains static .a library: python3-numpy path '/usr/lib/python3.13/site-packages/numpy/_core/lib/libnpymath.a' [staticdev]
ERROR: python3-numpy-2.1.3-r0 do_package_qa: QA Issue: File /usr/lib/python3.13/site-packages/numpy/__pycache__/__config__.cpython-313.pyc in package python3-numpy contains reference to TMPDIR [buildpaths]
ERROR: python3-numpy-2.1.3-r0 do_package_qa: Fatal QA errors were found, failing task.
ERROR: Logfile of failure stored in: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_package_qa.39587
ERROR: Task (/home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_package_qa) failed with exit code '1'
NOTE: Tasks Summary: Attempted 2118 tasks of which 2080 didn't need to be rerun and 1 failed.

Summary: 1 task failed:
  /home/tgamblin/workspace/yocto/poky/meta/recipes-devtools/python/python3-numpy_2.1.3.bb:do_package_qa
    log: /home/tgamblin/workspace/yocto/poky/build/tmp/work/core2-64-poky-linux/python3-numpy/2.1.3/temp/log.do_package_qa.39587
Summary: There were 4 ERROR messages, returning a non-zero exit code.
```

This is a common problem known as "host contamination", where for
various reasons (e.g. build files carrying the full path to a compiler
binary), the final copies of binaries, libraries, and other files to be
deployed contain references to the contents of the system they were
created on. This sort of issue conflicts with one of the key goals of
the Yocto Project, which is to support reproducible builds. There are a
lot of reasons for doing this, so it's probably better to read about
them on the [Yocto docs
page](https://docs.yoctoproject.org/test-manual/reproducible-builds.html).

Fixes for host contamination related to `TMPDIR` can vary a lot from
recipe to recipe, but they often involve some sort of post-compile
stripping of the paths from relevant files. In `python3-numpy`'s case,
we have to add a `do_install:append()` function to the recipe to strip
various build paths from the `__config__.py` file and recompile it:

```
+ # Remove references to buildpaths from numpy's __config__.py
+ do_install:append() {
+     sed -i \
+         -e 's|${S}=||g' \
+         -e 's|${B}=||g' \
+         -e 's|${RECIPE_SYSROOT_NATIVE}=||g' \
+         -e 's|${RECIPE_SYSROOT_NATIVE}||g' \
+         -e 's|${RECIPE_SYSROOT}=||g' \
+         -e 's|${RECIPE_SYSROOT}||g' ${D}${PYTHON_SITEPACKAGES_DIR}/numpy/__config__.py
+
+     nativepython3 -mcompileall -s ${D} ${D}${PYTHON_SITEPACKAGES_DIR}/numpy/__config__.py
+ }
+
```

If you look at the recipe closely, you'll also see that the
`FILES:${PN}-staticdev` variable still uses the old `core` (not `_core`)
path, so update that too:

```
~ FILES:${PN}-staticdev += "${PYTHON_SITEPACKAGES_DIR}/numpy/_core/lib/*.a \
+                           ${PYTHON_SITEPACKAGES_DIR}/numpy/random/lib/*.a \
+ "
```

### Success (Sort of)

Finally, bitbake succeeds:

```
tgamblin@megalith ~/workspace/yocto/poky/build ((HEAD detached at edfab1a3ceb))$ bitbake python3-numpy
Loading cache: 100% |##########################################################################################################################################################################################################################| Time: 0:00:00
Loaded 1969 entries from dependency cache.
Parsing recipes: 100% |########################################################################################################################################################################################################################| Time: 0:00:00
Parsing of 993 .bb files complete (992 cached, 1 parsed). 1969 targets, 56 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.9.1"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "x86_64-poky-linux"
MACHINE              = "qemux86-64"
DISTRO               = "poky"
DISTRO_VERSION       = "5.1"
TUNE_FEATURES        = "m64 core2"
TARGET_FPU           = ""
meta
meta-poky
meta-yocto-bsp
meta-selftest        = "HEAD:edfab1a3cebc454e4d63caae0134e02d169260c6"

Sstate summary: Wanted 17 Local 10 Mirrors 0 Missed 7 Current 730 (58% match, 99% complete)##############################################################################################################################                      | ETA:  0:00:00
Removing 1 stale sstate objects for arch qemux86_64: 100% |####################################################################################################################################################################################| Time: 0:00:00
Removing 5 stale sstate objects for arch core2-64: 100% |######################################################################################################################################################################################| Time: 0:00:00
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 2119 tasks of which 2109 didn't need to be rerun and all succeeded.
```

At this point you *could* summarize your changes and submit them, but
there's more due diligence that you should ideally do first as a
maintainer:

1. This initial update and build targeted the default `x86_64`
architecture, and we want to ensure it works for others (especially
since we modified a patch for RISCV32 support). There's a handy script
available in oe-core/poky called `buildall-qemu` that will take some
time (and space!) to complete, but it'll work through every supported
`qemu` architecture and LIBC (i.e. `glibc` and `musl`) combo, trying to
build the target for them and reporting the outcome. You can invoke it
for python3-numpy by doing `buildall-qemu python3-numpy`. When it's
done, you should find a file named something like
`python3-numpy-buildall.log` in your `build` directory with a summary
that you can include in your commit message.

2. Reproducibility tests! The setup details can be found
[here](https://docs.yoctoproject.org/test-manual/reproducible-builds.html#can-we-prove-the-project-is-reproducible).
This is another step that will take a long time, but it's important to
do to ensure that one of the project maintainers doesn't reject your
patch after it reaches the Autobuilder for integration testing (and it
makes them feel better about taking your patches!).

3. Generally, it's a good idea to some sort of actual runtime testing.
For most recipes, you can simply do something like `IMAGE_INSTALL:append
= "python3-numpy"` to your `local.conf` file and build
`core-image-minimal`, then run it with `runqemu kvm nographic` (the
`kvm` part assumes you have access to that on your build host). At this
point you can login and do some sort of simple check - in most Python
modules' case, you can just start the interpreter with `python3` and
then trying `import <package>`, which at bare minimum tests that the
recipe already pulls in all of the module's other dependencies. If your
recipe supports ptests (`ptest` should be in the `inherit` line), then
you can instead do `bitbake core-image-ptest-<recipename>` and run that,
then run `ptest-runner <recipename>` and see how those results come out.

## Post-Submission Surprises

If you were looking closely in the [Setup](/posts/yocto-maintainer-numpy-casestudy/#setup) section, you might
have noticed that there was a separate commit for adding `pkgconfig` to
numpy:

```
0b625064112 python3-numpy: inherit pkgconfig
```

At some point while readying my upgrade patch for submission, I
accidentally stripped that addition out of the recipe, and the patch
went upstream without it. This caused a fair bit of confusion once it
hit testing on the Yocto Autobuilder, as it only seemed to fail on
certain hosts such as Fedora 41, and I couldn't reproduce it locally.
Simply submitting the patch to add `pkgconfig` again was part of the
solution, but doing so alone only masked the real issue at play. It's
probably better for you to read the [Bugzilla
entry](https://bugzilla.yoctoproject.org/show_bug.cgi?id=15672) for more
details on the problem and its solution, but the message here is to
understand that even if you are doing your part and testing the changes
you make as much as possible, the Yocto Project is a complex system, and
even the best-tested upgrades can cause strange and challenging problems
to occur once they are introduced to the wider ecosystem.


## Conclusion

This has hopefully been an informative and interesting exploration into
some of the obstacles that can arise when trying to upgrade package
recipes for the Yocto Project. Usually, they're a lot simpler than this,
but I found this particular upgrade to be a great example because it
touches on so many different build issues at once. If you'd like to see
the final version of the upgrade as it is in the upstream repositories,
you can find it
[here](https://git.yoctoproject.org/poky/commit/?id=ea8594d4a0bf2e6f52372689b80fc00ce9cc7243).

It's worth noting that this walkthrough covers only the **final**
version of the `python3-numpy` upgrade from `v1.26.4` and `v2.1.3`.
There were previous attempts (now found only in the [oe-core mail
archive](https://lists.openembedded.org/g/openembedded-core/search?p=created%2C%2C%2C20%2C2%2C0%2C0&q=numpy))
to `v2.0.1` and `v2.1.2`, along with various related changes to recipes
like `piglit` happening around the same time. The content here doesn't
quite capture the amount of effort and discussion that ultimately went
into the upgrade, which goes to show how complicated maintaining the
project can really be sometimes when you extrapolate that to the entire
core layer - and that's just one of many.

This may be the last Yocto-focused post for a while, although some of
the concepts may feature in future writing about build/test automation
and making use of development boards such as the
[BeaglePlay](https://www.beagleboard.org/boards/beagleplay).

## See Also

- [Poky Repo](https://git.yoctoproject.org/poky/)
- [Reproducible Builds Project](https://reproducible-builds.org/)
- [NumPy Repo](https://github.com/numpy/numpy)
