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
    | +-(files)
    |   |
    |   +-(unpacked)
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

Our proposed package subdirectory structure looks like this:

Before running "make all":

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
    | +-(files)
    |
    +-(OPSI)
    | |
    | +-control
    |
    +-(files)
    |
    +-(build)

Note that there is no longer a directory "unpacked". See below for why.


After running "make all":

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
    | +-(files)
    |
    +-(OPSI)
    | |
    | +-control
    |
    +-(files)
    |
    +-(build)
      |
      +-(CLIENT_DATA)
      | |
      | +-(files) # See Note
      | |
      | +-delsub*.ins
      | |
      | +-setup*.ins
      | |
      | +-unins*.ins
      |
      +-newpackage.opsi

Note: 
* This directory will now be used as a target for unpacking, if required. No more files/unpacked.
* In theory, it would be possible to do away with this folder, and save/unpack the files from ./files to ./build/CLIENT_DATA. However, keeping OPSI scripts and installer files separate seems like a good idea (avoids naming conflicts, OPSI script files are easier to spot, etc.)

While there will most likely be packages that do not honor this HOWTO, new packages should be built according to it, and older ones changed at the next convenient moment.

