# New-Packages-HOWTO

# THIS IS A DRAFT - waiting for comments from other contributors

A bit of history:

There are/were several projects that offered both binary downloads of OPSI packages, as well as build recipes that make use of the "make" command and a corresponding file named "Makefile".

Of those offering build recipes, most looked like this:

(view of the package subdirectory)

    +-Makefile
    |
    +-md5sums.txt
    |
    +-(CLIENT_DATA)
    | |
    | +-delsub*.ins
    | |
    | +-setup*.ins
    | |
    | +-unins*.ins
    | |
    | +-(files) # optional
    |   |
    |   +-(unpacked) # optional
    |
    +-(OPSI)
      |
      +-control

Everything in (brackets) is a folder name, names without brackets are files.
The folders "files" and "unpacked" and the file "md5sums.txt" are optional.
The final *.opsi package ended up in the root of the package subdirectory, i.e. where the files "Makefile" and, if present, "md5sums.txt" reside.

The downside of this approach is that for every update, you need to make changes in several files:
* Makefile - new version number, possibly new download URL/file name
* CLIENT_DATA/setup*.ins - possibly new setup file name
* CLIENT_DATA/delsub.ins - possibly new uninstaller file name or UUID
* OPSI/control - new version number, changelog

Our idea, thus, is to simplify and centralize this, by adding a new folder "build", like it is common in the Unix world when building software using "make". That way, changes to those files can be made by scripts/commands within the Makefile itself.

The catch:
* The regular approach of simply writing the results to the ./build directory was never designed for large binary blobs. Blindly copying everything over means tripling (source, destination, inside package) the required space instead of doubling (source, inside package) it.
* Hardlinks come in handy for this, but special care must be taken that files that need to be edited in some way are not hardlinked, else they will be changed in the source directory as well - this would be especially bad for packages that rely on md5sums.txt to work.

Our proposed package subdirectory structure looks like this (incorporating changes that OPSI-Scripts should now end in .opsiscript, rather than .ins):

Before running "make all":

    +-(build) # will be created by first call of "build", and wiped by "clean" or "distclean"
    |
    +-(buildscripts)
    | |
    | +-script files that might be needed to apply patches during the build process
    |
    +-(CLIENT_DATA.in)
    | |
    | +-delsub*.opsiscript.in
    | |
    | +-setup*.opsiscript.in
    | |
    | +-unins*.opsiscript.in
    |
    +-(files) # optional
    |
    +-Makefile
    |
    +-md5sums.txt
    |
    +-(OPSI.in)
      |
      +-control.in

Note that there is no longer a directory "unpacked". See below for why.


After running "make all":

    +-(build)
    | |
    | +-(CLIENT_DATA)
    | | |
    | | +-(files) # See Note
    | | |
    | | +-delsub*.opsiscript
    | | |
    | | +-setup*.opsiscript
    | | |
    | | +-unins*.opsiscript
    | |
    | +-(OPSI)
    |   |
    |   +-control
    |
    +-(buildscripts)
    | |
    | +-script files that might be needed to apply patches during the build process
    |
    +-(CLIENT_DATA.in)
    | |
    | +-delsub*.opsiscript.in
    | |
    | +-setup*.opsiscript.in
    | |
    | +-unins*.opsiscript.in
    |
    +-(files) # optional
    |
    +-Makefile
    |
    +-md5sums.txt
    |
    +-(OPSI.in)
      |
      +-control.in

The resulting newpackage.opsi (and the newpackage.opsi.uploaded marker) will end up in the directory *above* the one where the Makefile is located. This is different from most other OPSI-related package build scripts, but follows standard Make conventions.

Note: 
* This directory will now be used as a target for unpacking, if required. No more files/unpacked.
* In theory, it would be possible to do away with this folder, and save/unpack the files from ./files to ./build/CLIENT_DATA. However, keeping OPSI scripts and installer files separate seems like a good idea (avoids naming conflicts, OPSI script files are easier to spot, etc.)

All *.in-files are templates.  If a particular *.opsiscript file does not need to be patched, drop the .in from the file name and it will be copied over untouched during the build.  
Inside a template file, please use @@@UNIQUEPLACEHOLDER@@@ for parts that need to be patched.

Makefiles should implement the following commands:

* clean (wipes "build", also wipes "files" for open-source packages)
* distclean (AKA purge, wipes "build" and "files", even for closed-source packages)
* get-orig-source (downloads open-source packages and checks their signature/hash, if available; checks MD5 of ./files/* in case of closed-source packages)
* build (starts the actual build process and creates the *.opsi file)
* install (tells the OPSI server to install the particular *.opsi file and asks for default parameters, if required)

Internally, the following sub-command names should be used:
* download (for open-source packages, as part of get-orig-source)
* unpack (optional, if downloaded packages need to be unzipped, untarred, ...)
* checkhashsig (optional, to check hashsums/signatures, if available)

While there will most likely be packages - and even Makefile templates - that do not honor this HOWTO, new packages should be built according to it, and older ones changed at the next convenient moment.  Updating the Makefile templates is a work in progress.

# Plans for the future

Unlike some other Projects providing Makefile-based OPSI package templates on Github, our Project uses one repository per package.
This follows "best git practice", but might make it inconvenient to install packages that depend on other packages.
For that, meta-repositories should be created, using "git submodule add", which pull in all the required repositories to build the package(s) in question.
One such meta-repository could be named "all-DotOP-open-source-packages".

A "supermake" Makefile then starts going through all subdirectories, calling in turn

* clean
* get-orig-source
* build
* install

when called with "make all"
