+++
title = "Maintaining the Yocto Project: Monitoring Recipes with AUH"
date = 2024-10-28
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

Depending on the particular recipes and the number of them to upgrade,
however, this could be quite tedious. Luckily, the Yocto Project
includes a tool called the Automatic Recipe Upgrade Helper (AUH) to
simplify this process. Combined with a CI system and a reasonably
powerful system with which to perform builds, the AUH can be used to
make the maintainer's recipe upgrade process smoother and less
repetitive.

## Setting up a Nightly AUH Job

### The Job Script

For this example, [Laminar CI](https://laminar.ohwg.net/) will be used
for setting up a nightly run, but your choice of CI system should work
if you're building Yocto images there already. Laminar jobs are just
Bash scripts marked as executable in `<laminar_home_dir>/cfg/jobs` and
with the `.run` extension (e.g. `auh.run`),

A very basic AUH job might look like this:

```
#!/bin/bash -ex

# clone poky and setup
git clone https://git.yoctoproject.org/poky
cd poky
source oe-init-build-env build-auh

# get AUH
git clone https://git.yoctoproject.org/auto-upgrade-helper
mkdir upgrade-helper

# copy the customized upgrade-helper.conf config file
cp /workspace/laminar/yoctocache/upgrade-helper.conf upgrade-helper

# build
./auto-upgrade-helper/upgrade-helper.py all
```

### Customizing AUH

Note this line from the script above:

```
cp /workspace/laminar/yoctocache/upgrade-helper.conf upgrade-helper
```

`upgrade-helper.conf` is a configuration file specifically for AUH. By
default, AUH will try to upgrade everything, which is probably too broad
a scope for your needs (and your build machine). One of the key
overrides to set in there is `maintainers_whitelist`, a space-separated
list of maintainers' email addresses to check against `maintainers.inc`
and attempt upgrades for matching recipes. For example, if you wanted to
automate the building of any recipes without assigned maintainers in
your AUH job, you might put the following in upgrade-helper.conf:

```
maintainers_whitelist=unassigned@yoctoproject.org
```

If you are listed as a maintainer for any recipes, you could also check
yours by instead setting

```
maintainers_whitelist=unassigned@yoctoproject.org yourname@youremail.com
```

### Periodic Execution

Now it's a good time to actually make it work. Trigger the nightly AUH
script with cronjob in a format like so:

```
0 22 * * * LAMINAR_REASON="Daily AUH build" laminarc queue auh
```

(remember that you should replace `auh` with whatever the name is for your
job script)

In this case, it's scheduled to run on the 22nd hour of every day. Even
with the scope of your AUH job narrowed down, it's probably still a good
idea to have it run overnight to avoid bogging down your system while
you use it.

When you log into the Laminar web UI and view the output logs from the
latest AUH run, you'll see a handy summary of the AUH run (just like you
had run it manually):

```
INFO: Generating work tarball in /auto/laminar/run/auh/162/poky/build-auh/upgrade-helper/20240524220028.tar.gz ...
INFO: Recipe upgrade statistics:

    * Succeeded: 8
        bind, 9.18.27, Unassigned <unassigned@yoctoproject.org>
        dhcpcd, 10.0.8, Unassigned <unassigned@yoctoproject.org>
        libslirp, 4.8.0, Unassigned <unassigned@yoctoproject.org>
        python3-setuptools, 70.0.0, Unassigned <unassigned@yoctoproject.org>
        libxcb, 1.17.0, Unassigned <unassigned@yoctoproject.org>
        xserver-xorg, 21.1.13, Unassigned <unassigned@yoctoproject.org>
        xwayland, 24.1.0, Unassigned <unassigned@yoctoproject.org>
        wireless-regdb, 2024.05.08, Unassigned <unassigned@yoctoproject.org>
    * Failed (devtool error): 3
        ovmf, edk2-stable202405, Unassigned <unassigned@yoctoproject.org>
        libgit2, 1.8.1, Unassigned <unassigned@yoctoproject.org>
        pinentry, 1.3.0, Unassigned <unassigned@yoctoproject.org>
    * Failed(do_compile): 1
        cmake, 3.29.3, Unassigned <unassigned@yoctoproject.org>

    TOTAL: attempted=12 succeeded=8(66.67%) failed=4(33.33%)

Recipe upgrade statistics per Maintainer:

    Unassigned <unassigned: attempted=12 succeeded=8(66.67%) failed=4(33.33%)
```

From here you can use the information and build results to get the
upgrades upstream as required.

## Conclusion

What I've described is a relatively low-effort way to automate tracking
of recipe upgrades, which is one of the best places to contribute to
Yocto. However, there are lots of little steps that could be taken to
improve this process:

- proper sstate/workspace reuse in the AUH job script and in CI setup
- Archiving the result tarballs
- Configuring email so AUH can send out the results to the user - **not
  the official mailing list, because there's already a service doing
  that!**
- Adding testimage and other runtime checks

In the next installment of this series, I'll discuss what to do with a
recipe whose AUH upgrade failed.

## See Also

- [Web archive of Auto Upgrade Helper mail for the OE-Core layer](https://lists.openembedded.org/g/openembedded-core/search?q=posterid:4563390)
- [Auto Upgrade Helper Repo](https://git.yoctoproject.org/auto-upgrade-helper/)
- [Poky Repo](https://git.yoctoproject.org/poky/)
- [Laminar CI](https://laminar.ohwg.net/)
