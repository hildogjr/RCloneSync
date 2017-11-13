# RCloneSync


Python 2.7 cloud sync utility using rclone

[Rclone](https://rclone.org/) provides a programmatic building block interface for transferring files between a cloud service 
provider and your local filesystem (actually a lot of functionality), but rclone does not provide a turnkey bidirectional 
sync capability.  RCloneSync.py provides a bidirectional sync solution using rclone.

I use RCloneSync on a Centos 7 box to sync both Dropbox and Google Drive to a local disk which is Samba shared on my LAN. 
I run RCloneSync as a Cron job every 30 minutes, or on-demand from the command line.  
RCloneSync was developed and debugged for Google Drive and Dropbox (not tested on other services).  

### High level behaviors / operations
-  Keeps `rclone lsl` file lists of the Local and Remote systems, and on each run checks for deltas on Local and Remote
-  Applies Remote deltas to the Local filesystem, then `rclone syncs` the Local to the Remote filesystem
-  Handles change conflicts nondestructively by creating _LOCAL and _REMOTE file versions
-  Reasonably fail safe:
	- Lock file prevents multiple simultaneous runs when taking a while
	- File access health check using `RCLONE_TEST` files (see --CheckAccess switch)
	- Excessive deletes abort - Protects against a failing `rclone lsl` being interpreted as all the files were deleted.  See `maxDelta` within the code and --Force switch
	- If something evil happens, RCloneSync goes into a safe state to block damage by later run.  (See **Runtime Error Handling**, below)


```
xxx@xxx RCloneSyncWD]$ ./RCloneSync.py -h
2017-11-12 14:49:04,799/WARNING:  ***** BiDirectional Sync for Cloud Services using RClone *****
usage: RCloneSync.py [-h] [--FirstSync] [--CheckAccess] [--Force]
                     [--ExcludeListFile EXCLUDELISTFILE] [--Verbose]
                     [--DryRun]
                     {Dropbox:,GDrive:} LocalRoot

***** BiDirectional Sync for Cloud Services using RClone *****

positional arguments:
  {Dropbox:,GDrive:}    Name of remote cloud service
  LocalRoot             Path to local root

optional arguments:
  -h, --help            show this help message and exit
  --FirstSync           First run setup. WARNING: Local files may overwrite
                        Remote versions. Also asserts --Verbose.
  --CheckAccess         Ensure expected RCLONE_TEST files are found on both
                        Local and Remote filesystems, else abort.
  --Force               Bypass maxDelta (50%) safety check and run the sync.
                        Also asserts --Verbose.
  --ExcludeListFile EXCLUDELISTFILE
                        File containing rclone file/path exclusions (Needed
                        for Dropbox)
  --Verbose             Event logging with per-file details (Python INFO level
                        - default is WARNING level)
  --DryRun              Go thru the motions - No files are copied/deleted.
                        Also asserts --Verbose.
```	

Typical run log:
```
./RCloneSync.py Dropbox:  /mnt/raid1/share/public/DBox/Dropbox --CheckAccess --ExcludeListFile /home/xxx/RCloneSyncWD/Dropbox_Excludes --Verbose

2017-10-29 13:05:19,556/WARNING:  ***** BiDirectional Sync for Cloud Services using RClone *****
2017-10-29 13:05:19,565/INFO:  >>>>> Checking rclone Local and Remote access health
2017-10-29 13:05:24,001/INFO:  >>>>> Generating Local and Remote lists
2017-10-29 13:05:28,534/INFO:  LOCAL    Checking for Diffs                  - /mnt/raid1/share/public/DBox/Dropbox
2017-10-29 13:05:28,535/INFO:  LOCAL      File is new                       - Public/imagesCASWV071 - Copy.jpg
2017-10-29 13:05:28,535/WARNING:       1 file change(s) on the Local system /mnt/raid1/share/public/DBox/Dropbox
2017-10-29 13:05:28,535/INFO:  REMOTE   Checking for Diffs                  - Dropbox:
2017-10-29 13:05:28,535/INFO:  REMOTE     File was deleted                  - Public/imagesCAB79VH9a.jpg
2017-10-29 13:05:28,536/WARNING:       1 file change(s) on Dropbox:
2017-10-29 13:05:28,536/INFO:  >>>>> Applying changes on Remote to Local
2017-10-29 13:05:28,536/INFO:  LOCAL      Deleting file                     - "/mnt/raid1/share/public/DBox/Dropbox/Public/imagesCAB79VH9a.jpg" 
2017-10-29 13:05:28,542/INFO:  >>>>> Synching Local to Remote
2017/10/29 13:05:28 INFO  : Dropbox root '': Modify window is 1s
2017/10/29 13:05:30 INFO  : Public/imagesCASWV071 - Copy.jpg: Copied (new)
2017/10/29 13:05:32 INFO  : Dropbox root '': Waiting for checks to finish
2017/10/29 13:05:32 INFO  : Dropbox root '': Waiting for transfers to finish
2017/10/29 13:05:32 INFO  : Waiting for deletions to finish
2017/10/29 13:05:32 INFO  : 
Transferred:   7.925 kBytes (1.833 kBytes/s)
Errors:                 0
Checks:               851
Transferred:            1
Elapsed time:        4.3s

2017-10-29 13:05:32,867/INFO:  >>>>> rmdirs Remote
2017-10-29 13:05:36,902/INFO:  >>>>> rmdirs Local
2017-10-29 13:05:36,911/INFO:  >>>>> Refreshing Local and Remote lsl files
2017-10-29 13:05:41,488/WARNING:  >>>>> All done.
```
Note the two different styles of timestamps in the above log.  Turning on --Verbose in RCloneSync also turns on rclone's --verbose only 
for the `rclone sync` operation.

## RCloneSync Operations

RCloneSync keeps copies of the prior sync file lists of both local and remote, and on a new run checks for any changes 
locally and then remotely.  Note that on some (all?) cloud storage systems it is not possible to have file timestamps 
that match between the local and cloud copies of a file.  RCloneSync works around this problem by tracking local-to-local 
and remote-to-remote deltas, and then applying the changes on the other side. 

### Notable features / functions / behaviors

- RCloneSync applies any changes to the Local file system first, then uses `rlone sync` to make the Remote filesystem match the Local.
In the tables below, understand that the last operation is to do an `rclone sync` if RCloneSync makes changes on the local filesystem.

- Any empty directories after the RCloneSync are deleted on both the Local and Remote filesystems.

- **--FirstSync** - This will effectively make both Local and Remote contain a matching superset of all files.  Remote 
files that do not exist locally will be copied locally, and the process will then sync the Local tree to the Remote.  

- **--CheckAccess** - Access check files is an additional safety measure against data loss.  RCloneSync will ensure it can 
find matching RCLONE_TEST files in the same places in the local and remote file systems.  Time stamps and file contents 
are not important, just the names and locations.  Place one or more RCLONE_TEST files in the local or remote filesystem and then 
do either a run without --CheckAccess or a --FirstSync to set matching files on both filesystems.

- **Runtime Error Handling** - Certain RCloneSync critical errors, such as not able to `rclone lsl` the local or remote targets, 
will result in an RCloneSync lockout.  The recovery is to do a --FirstSync again.  It is recommended to use --FirstSync 
--DryRun initially and carefully review what changes will be made before running the --FirstSync without --DryRun.  
Most of these events come up due to rclone returning a non-zero status from a command.  On such a critical error 
the *_localLSL and *_remoteLSL files are renamed adding _ERROR, which blocks any future RCloneSync runs (since the 
original files are not found).  These files may possibly be valid and may be renamed back to the non-_ERROR versions 
to unblock further RCloneSync runs.  Some errors are considered temporary, and re-running the RCloneSync is not blocked.  

- **Lock file** - When RCloneSync is running, a lock file is created (/tmp/RCloneSync_LOCK).  If RCloneSync should crash or 
hang the lock file will remain in place and block any further runs of RCloneSync.  Delete the lock file as part of 
debugging the situation.  The lock file effectively blocks follow on CRON scheduled runs when the prior invocation 
is taking a long time.  The lock file contains the job command line and time, which may help in debug.


### Usual sync checks

 Type | Description | Result| Implementation ** 
--------|-----------------|---------|------------------------
Remote new| File is new on remote, does not exist on local | Remote version survives | `rclone copyto` remote to local
Remote newer| File is newer on remote, unchanged on local | Remote version survives | `rclone copyto` remote to local
Remote deleted | File is deleted on remote, unchanged locally | File is deleted | `rclone delete` local
Local new | File is new on local, does not exist on remote | Local version survives | `rclone sync` local to remote
Local newer| File is newer on local, unchanged on remote | Local version survives | `rclone sync` local to remote
Local older| File is older on local, unchanged on remote | Local version survives | `rclone sync` local to remote
Local deleted| File no longer exists on local| File is deleted | `rclone sync` local to remote


### *UNusual* sync checks

 Type | Description | Result| Implementation **
--------|-----------------|---------|------------------------
Remote new AND Local new | File is new on remote AND new on local | Files renamed to _LOCAL and _REMOTE | `rclone copyto` remote to local as _REMOTE, `rclone moveto` local as _LOCAL
Remote newer AND Local changed | File is newer on remote AND also changed (newer/older/size) on local | Files renamed to _LOCAL and _REMOTE | `rclone copyto` remote to local as _REMOTE, `rclone moveto` local as _LOCAL
Remote newer AND Local deleted | File is newer on remote AND also deleted locally | Remote version survives  | `rclone copyto` remote to local
Remote deleted AND Local changed | File is deleted on remote AND changed (newer/older/size) on local | Local version survives |`rclone sync` local to remote
Local deleted AND Remote changed | File is deleted on local AND changed (newer/older/size) on remote | Remote version survives  | `rclone copyto` remote to local

** If any changes are made on the Local filesystem then the final operation is an `rclone sync` to update the Remote filesystem to match.

### Unhandled

 Type | Description | Comment 
--------|-----------------|---------
Remote older|  File is older on remote, unchanged on local | `rclone sync` will push the newer local version to the remote.
Local size | File size is different (same timestamp) | Not sure if `rclone sync` will pick up on just a size difference and push the local to the remote.


