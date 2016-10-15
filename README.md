![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)
==============

cppcryptfs
------

This software is based on the design of [gocryptfs](https://github.com/rfjakob/gocryptfs), an encrypted overlay filesystem written in Go.

cppcryptfs is an implementation of the gocryptfs filesystem in C++ for Windows.

It uses the the [Dokany](https://github.com/dokan-dev/dokany) driver and library to provide a virtual fileystem in user mode under Windows.


Current Status
--------------

cppcryptfs is pre-alpha, or more accurately: EXPERIMENTAL.

Testing
-------

cppcryptfs seems to work.  It passes 171/171 of the tests in [winfstest](https://github.com/dimov-cz/winfstest).


Build Requirements
-----
	
	Microsoft Visual Studio 2015 Community Edition
	OpenSSL - https://github.com/openssl/openssl (static build recommended)
	rapidjson - https://github.com/miloyip/rapidjson (for parsing gocryptfs.conf)
	Dokany - https://github.com/dokan-dev/dokany

	For Dokany, you probably want to use the binary distribution from here:
		https://github.com/dokan-dev/dokany/releases
	(be sure to select "install development files" in the installer options)



There are detailed build instructions in [INSTALL.md](INSTALL.md).


Use
-------

cppcryptfs doesn't require administrator privileges to run, but
if it is not running as administrator, then it won't be able
to acquire the SE_SECURITY_NAME privilege.  SE_SECURITY_NAME is
needed for reading all of the security attributes of files.

cppcryptfs seems to work without SE_SECURITY_NAME.  If you do
run into problems, then you can try running cppcryptfs as adminstrator and see if that helps.

To make a new encrypted virtual fileystem, first click the "Create" tab.

![Alt text](/screenshots/screenshot_create.png?raw=true "Create tab")

You need to find or create (you can create a directory in the directory selector in the UI) an empty directory to be the root of your filesystem.

It is strongly recommended that this directory reside on an NTFS filesystem.

Then you need to choose a (hopefully strong) password and repeat it.  The dialog box will accept at most 255 characters for the password.

You can choose to have your file names encryped using AES256-EME or not to encrypt the file names (plain text).

When you click on the "Create" button, a gocryptfs.conf file will be created in the directory.  Unless you choose to use plain text file names, a gocryptfs.diriv will also be created there.  Be sure to backup these files in case they get lost or corrupted.  You won't be able to access any of your data if something happens to gocryptfs.conf.  gocryptfs.conf will never change for the life of your filesystem unless you change the volume label (see bellow).

If you choose to give the volume a label, then the label will be encrypted in gocryptfs.conf.  The maximum volume label length is 32 characters. 

The volume label is AES256-GCM encrypted using the master key and a 128-bit random initialization vector and 8 zero bytes of auth data.  Then it is base64 encoded along with the initilization vector and saved in gocryptfs.conf.

The volume label feature is the only cryptographic addition to cppcryptfs that is not part of the cryptographic design of gocryptfs.  If you do not trust that it was designed correctly, then don't use it.

You can right click on the mounted drive letter in File Explorer, select "Properties", and change the volume label.  However, doing so will cause cppcryptfs to re-write gocryptfs.conf when the drive is dismounted. This does entail some risk to your gocryptfs.conf.  Again, it's a good a idea to backup your gocryptfs.conf file somewhere.  

Then go to the "Mount" tab and select a drive letter and select the folder you
just created the filesystem in.  Then enter the password and click on the "Mount" button.

![Alt text](/screenshots/screenshot_mount.png?raw=true "Mount tab")

Your will then have a new drive letter, and you can use it like a normal drive letter and store your sensitive information there.  The data is encrypted and saved in files in the folder you specified.

The file data is encrypted using AES256-GCM.

For technical details of the cryptographic design of gocryptfs, please visit
the [gocryptfs project page](https://github.com/rfjakob/gocryptfs).


When you are finished using the drive letter, go to the "Mount" tab and select the drive letter and click on "Dismount" or click "Dismount All".  The drive letter(s) will be dismounted, and the encryption keys will be erased from memory. 

You can mount as many gocryptfs filesystems as you have unused drive letters available.

Passwords and keys are locked in memory using VirtualLock(). When they are no longer needed, they are erased using SecureZeroMemory() and then unlocked.  If you never hibernate your computer, then you don't have to worry about your passwords or keys ever being written to the hard drive.

If you close the cppcryptfs window, then it will hide itself in the system tray. To exit cppcryptfs, use the Exit button on the mount page or the context menu of the system tray icon.


Command Line Options
----
cppcryptfs accepts some command line options for mounting and umounting filesystems.  Currently, filesystems can be created only by using the gui.

Passwords passed through the command line are not really secure.  cppcryptfs locks and zeros its internal copies of the command line, but it does not zero the command line stored in the Windows PEB (Process Environment Block). Also, it is unkown how many times the command line might be copied by Windows out of cppcryptfs' control.  So there is some chance that a password passed via the command line might end up in the paging file.

```
usage: cppcryptfs [OPTIONS]

Mounting:
  -m, --mount=PATH              mount filesystem locate at PATH
  -d, --drive=D                 mount to drive letter D
  -p, --password=PASSWORD       use password PASSWORD

Unmounting:
  -u, --unmount=D               umount drive letter D
  -u, --umount=all              unmount all drives

Misc:
  -t, --tray                    hide in system tray
  -x, --exit                    exit if no drives mounted
  -h, --help                    display this help message

```

Note: when using the short version of the option, you should not use the equal sign between the option and its argument.  When using the long version of the option, the equal sign is optional. e.g. these will work

```
cppcryptfs -m c:\tmp\test -d k -p XYZ
cppcryptfs --mount=c:\tmp\test --drive=k --password=XYZ
cppcryptfs --mount c:\tmp\test --drive k --password XYZ

```
cppcryptfs is a Windows gui application and not a console application.  However, when started with command line options, it will try to write any error messages to the console (if any) that started it.

There can be only one instance of cppcryptfs running at any time.

When cppcryptfs is invoked, then it checks to see if there is another instance running.  If there is, then if there are no command line options, the second instance of cppcryptfs will simply exit.

If the second instance is invoked with command line options, it will send its command line to the already-running instance using the WM_COPYDATA message.  It will block until the already-running instance has processed the command line and then exit.  Any error messages that result from processing the command line will be printed in the cmd window in which the second instance was invoked.

Therefore, if you plan to use cppcryptfs in batch files, you need to start an instance in the background first.  Then you should do the other operations in the foreground so they will block until completed.

If you start "cppcryptfs --tray" in the background, then if there is already a running instance, then that instance will be told to hide itself in the system tray.  If there is not already an instance running, then you will have started cppcryptfs hidden in the system tray, running in the background. 

Here is an example Windows cmd batch file using cppcryptfs.


```
@rem ====================================================
@rem run cppcryptfs in background and give it time to start up
@rem ====================================================

start cppcryptfs.exe --tray
timeout /t 1 >nul

@rem ====================================================
@rem Mount drive U:
@rem ====================================================

cppcryptfs.exe --mount=d:\TestCppCryptFS --drive=u --password=PASSWORD --tray  --exit

@rem ====================================================
@rem Mount drive V:
@rem ====================================================

cppcryptfs.exe --mount=d:\TestCppCryptFS2 --drive=v --password=PASSWORD --tray  --exit

@rem ====================================================
@rem Run any command with the mounted drives
@rem ====================================================

copy  C:\test.txt U:\test.txt
copy  C:\test.txt V:\test.txt
```

Here is an example cygwin bash scrypt.  Note that in bash, you need to
use double-backslashes in the mount paths.


```

#!/bin/bash
# start cppcryptfs in the background and hidden in the system tray
/cygdrive/c/bin/cppcryptfs -t &
# give it time to initialize
sleep 1
# mount a filesystem and wait for the mount operation to complete
/cygdrive/c/bin/cppcryptfs --mount c:\\tmp\\test -d k -p XYZ
# do backup operation
rsync .....
# unmount all drives and exit
/cygdrive/c/bin/cppcryptfs -u all -x

```


File name and path length limits
------

If "Long file names" (the default) is specfied when creating the fileystem, or if plain text file names are used, and if the filesystem is located on NTFS, then a file or directory name can be up to 255 characters long, and a full path can be up to 32,000 characters long.

If "Long file names" is not specified and plain text file names aren't used, then the maximum length of a file or directory name is 160 characters.  But the full path limit is still 32,000 characters (assuming NTFS).

When a file name is encrypted, it is converted from UNICODE-16 to UTF-8 which, depending the language, might cause the number of characters to increase.  Then it is encrypted, which causes it to be padded by up to 16 bytes. Then it is base64 encoded, which typically results in a 33% increase in length.  The encrypted file names can therefore be signifcantly longer than the unencrypted names.

Also, the path to the directory in which the encrypted fileystem resides must be pre-pended to the path of the encrypted file names.

Older filesystems, such as FAT32, will limit the total path length to 259 characters.

It is therefore strongly recommended to use NTFS whenever possible.

A lot of Windows progams, including File Explorer that comes with Windows, have problems with long paths.  If you use encrypted file names, then you might need to use a third-party file manager that handles long file paths if you want to move the root of your encrypted filesystem.  It's a good idea to copy it and then delete the old one instead of moving it in case your file manager has problems.


Case Sensitivity
-----
Windows filesystems are not case-sensitive, but they are case-preserving.  The gocryptfs filesystem with encrypted filenames is case-senstitive.

The way the file name encryption works means that if you create a file as Foo.txt and then try
to open it as foo.txt, it will not be found.  However, on a regular Windows filesystem, it would be found.  This is not normally a problem because files are usually opened using the same case that was
used when they were created.  So far, the only software that has been observed creating files with one case and then trying to open them with another case is Microsoft Visual Studio.

Currently, the only solution to this problem is to use plain text file names.

Performance
------
Below are some benchmark results.  The tests were conducted using the cygwin utilities under Windows 10 64-bit running on an Intel i5-4200U cpu with a Crucial M500 240GB ssd.  Encrypted file names and long file names were used with cppcryptfs.

Windows Defender realtime scanning was disabled during the tests because it really slows down cygwin tar.  It took 2m43.600s to extract linux-3.0.tar.gz on native NTFS with realtime scanning enabled.

cppcryptfs performs about the same as the mirror sample program from Dokany which doesn't do encryption.  The SSD is rated for 250 MB/sec streaming write performance.

```
								cppcryptfs		native NTFS		Dokany mirror
								
Streaming Write					164 MB/s		223 MB/s		173 MB/s
Extract linux-3.0.tar.gz		1m36.412s		0m21.291s		1m34.125s	
ls -lR linux-3.0				1m1.979s		0m2.983s		1m11.618s
Delete linux-3.0 				1m28.749s		0m10.144s		1m24.677s

```

Compatibility with gocryptfs
------

cppcryptfs can mount all filesystems created by gocryptfs v0.7 and higher. Likewise, filesystems created by cppcryptfs with "long file names = off" can be mounted by gocryptfs v0.7 and higher. Filesystems with "long file names = on" can mounted by gocryptfs v0.9 and higher.

The gocryptfs [compatability matrix](https://github.com/rfjakob/gocryptfs/wiki/Compatibility) provides more details. cppcryptfs *requires* the DirIV, EMENames and GCMIV128 feature flags. It *supports* LongNames and can create filesystems with the flag on and off.
