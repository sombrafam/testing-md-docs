# OpenStack SRU

Guide on how to create an SRU for patches of OpenStack projects. For
this example we are going to walk through the process to backport this Heat
Dashboard patch[1]. The customer in question is using Bionic/Stein and this
patch is already backported to the upstream target branch[its already in Train]. 
For OpenStack backports, the patch always needs to be backported upstream first, from the
master patch, until it lands to the target stable branch. Once its in the stable
branch, you can either create an SRU for that patch or wait the branch to be
tagged upstream, if not already, and request a point release for our OpenStack
Development team.

**Note: About point releases this can take a long time and point releases need to be coordinate with Openstack Dev Team, so ask ahead of time to make sure we're not going to conflicting versions.**

e.g. Fix a bug and gets approved so now we're at 1.5.1 
but if a point release is happening at the same time we could going to 2.0.0 
and miss our updated bug fix. (TODO: i probably didnt explain that very well )

[1914100](https://bugs.launchpad.net/ubuntu/+source/heat-dashboard/+bug/1914100)

### 1. Fix Merged Upstream First

Make sure that the fix you want is backported to the release you are targeting. Please refer to the 
[OpenDevDevelopment Guide](https://docs.opendev.org/opendev/infra-manual/latest/developers.html)
on how to do that if you don't know.

[opendev](https://review.opendev.org/c/openstack/heat-dashboard/+/694952)

[github](https://github.com/openstack/heat-dashboard/commit/837a40e3b7358b8ebf6a27de1bd9d904d98bb451)

### 2 Test Package without Fix

Create a reproducer (if possible) for the issue without adding fix.
If it is not possible to create a reproducer, bring it up during weekly meeting. 

### 3 Create Debian Patch

Cloud SRU for Openstack uses `pull-uca-source` for debian package maintenance. 

```sh
pull-uca-source python3-heat-dashboard stein
```

Once your backport is landed upstream on the target branch, you need to
download the code currently used in the ubuntu package and create a debian
package patch. You will need a local machine and a container, or VM where you
will build your package. The container/vm need to have the same Ubuntu series
you are targeting your package. Having a shared folder between them will help
you a lot.

On your local machine install the tools needed:

```sh
snap install cmadison
```

```sh
cmadison python3-heat-dashboard 
 python3-heat-dashboard | 1.5.0-0ubuntu3~cloud0         | stein             | bionic-updates  | all
 python3-heat-dashboard | 1.5.0-0ubuntu3~cloud0         | stein-proposed    | bionic-proposed | all
 python3-heat-dashboard | 2.0.0-0ubuntu1~cloud0         | train             | bionic-updates  | all
 python3-heat-dashboard | 2.0.0-0ubuntu1~cloud0         | train-proposed    | bionic-proposed | all
 python3-heat-dashboard | 3.0.0-0ubuntu0.20.04.1~cloud0 | ussuri            | bionic-updates  | all
 python3-heat-dashboard | 3.0.0-0ubuntu0.20.04.1~cloud0 | ussuri-proposed   | bionic-proposed | all
 python3-heat-dashboard | 4.0.0-0ubuntu1~cloud0         | victoria          | focal-updates   | all
 python3-heat-dashboard | 4.0.0-0ubuntu1~cloud0         | victoria-proposed | focal-proposed  | all
 python3-heat-dashboard | 4.0.0-0ubuntu1~cloud0         | wallaby           | focal-updates   | all
 python3-heat-dashboard | 4.0.0-0ubuntu1~cloud0         | wallaby-proposed  | focal-proposed  | all

```

In you local machine, clone the repo of the project your are working and
   create a patch for it.
   
```sh
git clone https://github.com/openstack/heat-dashboard.git  
```
There could also be a mirrored repo in opendev. Would use best judgement to see which repo is more up to date. 

```sh
git clone https://opendev.org/openstack/heat-dashboard.git
```

```sh
# clone original git repo 
git clone https://opendev.org/openstack/heat-dashboard.git
cd heat-dashboard/
git reflog
git format-patch -1 <git reflog #>
git format-patch -1 934c8e1bbae8794d71cf9fbf0a85c29f2b900e76
# creates a .patch file
0001-Run-npm-nodejs-job-with-Firefox-browser.patch
# usually we rename this to something like 
# (lp#goes first) <220045-run-npm-nodejs-job-with-Firefox-browser.patch>
# create new directory with lp# in ~ area
mkdir ~/1914100/
cd ~/1914100/
# clone debian source package 
pull-uca-source python3-heat-dashboard
```

### 4. Working with quilt

```
# cd to debian package
cd ~/220045/heat-dashboard-4.0.0/
# you can cp the import to ~/1914100/ directory makes it easier to grab if we messed up
quilt import ../1914100-run-npm-nodejs-job-with-Firefox-browser.patch 
Importing patch ../1914100-run-npm-nodejs-job-with-Firefox-browser.patch (stored as 1914100-run-npm-nodejs-job-with-Firefox-browser.patch)
quilt push 
quilt refresh
# check that code changes we're saved 
```
We will not be going over quilt in-depth. 
Please use this as a reference if you are stuck [quilt help](https://raphaelhertzog.com/2012/08/08/how-to-use-quilt-to-manage-patches-in-debian-packages/) 
```
# See the next step on what to fill when you run the dch command
dch -i
```

In this step, we are going to edit the debian changelog (dch) version file that is opened for us.
See that the version needs to be properly pushed. dch tries to do this automatically, but you might have to fix according to your case. In the case of our bug:

*NOTE: If you upload this to your personal PPA, you need a generic series name. NOT bionic-stein, just **bionic**.
```sh
heat-dashboard (1.5.0-0ubuntu3~cloud1) bionic-stein; urgency=medium

  * d/p/1914100-run-npm-nodejs-job-with-Firefox-browser.patch (LP: #1914100)
    - Switch to use Firefox instead of Chrome as default browser

 -- Heather Lemon <heather.lemon@canonical.com>  Mon,02 Feb 2021 15:52:57 -0700

```
### 5. Edit Quilt Header 
```sh
quilt header --dep3 -e
```
Follow the quilt header guidelines here: [quilt guidelines](https://dep-team.pages.debian.net/deps/dep3/)

**Note: Some upstream maintainers prefer the header remains untouched. So you may/may not have to do this step.**

### 6. Build Package

Now, jump to the container, that in our case will have all the changes to the
package and build it.

1. On your container, install the development tools and the build dependencies:

Also, uncomment all source repositories from update `/etc/apt/sources.list`
Example:

```sh
## Major bug fix updates produced after the final release of the
## distribution.
deb http://archive.ubuntu.com/ubuntu bionic-updates main restricted
deb-src http://archive.ubuntu.com/ubuntu bionic-updates main restricted

```

Install package dependencies:

```sh
sudo apt build-dep .
```

Build package

```sh
debuild -uc -us -S
```

Then generate the debdiff. 
old.dsc new.dsc > lp#bug-mypatch.debdiff 

```sh
cd ..
debdiff python3-heatdashboard-01.dsc python3-heatdashboard-01.dsc > 1914100-python3-heatdashboard.debdiff
```
### 7. Sign package with your private key

In order to test the package, let's sign and upload it to our private ppa. Back
to your local machine (were I assume you have all your signatures configured),
run the following:

```sh
debsign -k <key id> python3-heat-dashboard_source.changes
```
### 8. Upload to personal PPA
Package_source.changes file (must be signed before upload) 

**Note: If you are hanging during upload you might need to disconnect from the VPN if connected**
```sh
dput ppa:<launchpad_name>/heat-dashboard python3-heat-dashboard_source.changes
```

You know when the package has been successfully built by 

1. You get an email from launchpad (Accepted) 
2. View Package Details after successful upload
3. Build Status has a green checkmark 

### 9. Building New Changes 

Once the debian package is saved in launchpad ppa, we can pull the debian package from the ppa into the testing enviornment.

**Note: These commands are run from inside of the openstack-dashboard/0 container**

```sh
sudo add-apt-repository ppa:<username>/heat-dashboard
sudo apt-get update
# check the correct version number, should be latest
apt-cache policy python3-heat-dashboard
dpkg -l python3-heat-dashboard | cat
sudo apt-get install python3-heat-dashboard (latest)
```

### 10. Update LP Tag

Add tag to LP bug 
`sts-sru-needed`

Edit the description for SRU  https://wiki.ubuntu.com/StableReleaseUpdates 
```text
SRU Bug Template

[Impact] 

 * An explanation of the effects of the bug on users and

 * justification for backporting the fix to the stable release.

 * In addition, it is helpful, but not required, to include an
   explanation of how the upload fixes this bug.

[Test Case]

 * detailed instructions how to reproduce the bug

 * these should allow someone who is not familiar with the affected
   package to reproduce the bug and verify that the updated package fixes
   the problem.

[Where problems could occur]

 * Think about what the upload changes in the software. Imagine the change is
   wrong or breaks something else: how would this show up?

 * It is assumed that any SRU candidate patch is well-tested before
   upload and has a low overall risk of regression, but its important
   to make the effort to think about what 'could' happen in the
   event of a regression.

 * This must never be None or Low, or entirely an argument as to why
   your upload is low risk.

 * This both shows the SRU team that the risks have been considered,
   and provides guidance to testers in regression-testing the SRU.

[Other Info]
 
 * Anything else you think is useful to include
 * Anticipate questions from users, SRU, +1 maintenance, security teams and the Technical Board
 * and address these questions in advance
```

[PerformingSRUVerification](https://wiki.ubuntu.com/QATeam/PerformingSRUVerification)


### 11. SRU Verification 
Complete steps for reproducer and test in new environment that fix works. 


Add tag to LP bug
`verification-stein-done`

### 12. Add your bug to the SRU cloud review list under Next
[SRU Cloud Review List](https://docs.google.com/spreadsheets/d/1xsKUS6_4gpgwIH0-CGBw39KTndwSIvcMoKrQzr-dgxI/edit#gid=0)
