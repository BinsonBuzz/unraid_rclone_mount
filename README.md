# unraid_rclone_mount
scripts to create rclone vfs mounts on unraid to allow fast launch times with Plex (or Emby).

The main thread

https://forums.unraid.net/topic/75436-guide-how-to-use-rclone-to-mount-cloud-drives-and-play-files/

<b>Plugins needed</b>

Rclone beta – installs rclone and allows the creation of remotes and mounts
User Scripts – controls how mounts get created


<b>Optional Plugins</b>

Nerd Tools - used to install Unionfs which allows a 2nd mount to be created that merges the rclone mount with files locally e.g. new TV episodes that haven’t been uploaded yet, so that dockers like sonar, radar etc can see that you’ve already got the files and don’t try to add them to your library again.  In the future hopefully this will be replaced with rclone’s new Union allowing for an all-in-one solution

Optional: to allow an extra 750GB/day upload create 2 additional remotes to support a Team Drive

<li>tdrive: - a teamdrive remote.  Note: you need to use a different gmail/google account to the one above (which creates and shares the Team Drive) to create the token - any google account will do.  I recommend creating a 2nd client_id using this account</li>
<li>ve_media_vfs: - a crypt remote that is mounted locally and decrypts the encrypted files uploaded to the team drive</li>


I use a rclone vfs mount as opposed to a rclone cache mount as this is optimised for streaming, has faster media start times, and limits API calls to google to avoid bans.

<b>2.       Create Mountcheck files</b>

This blank file is used in the following scripts to verify if the mounts have been created properly.  Run these commands:

<b>3.      Mount script - see https://github.com/BinsonBuzz/unraid_rclone_mount for latest script

Create a new script in user scripts to create the rclone mount, unionfs mount and start dockers that need the mounts.  I run this script on a 10 min */10 * * * * schedule so that it automatically remounts if there’s a problem. 

The script:

Checks if an instance is already running
Update: Mounts rclone gdrive and tdrive remotes
Update: Mounts unionfs creating a 3-way union between rclone gdrive remote, tdrive remote and local files stored in /mnt/user/rclone_upload
Starts dockers that need the unionfs mount e.g. radarr
New: used rclone rc to populate the directory cache

I've tried to annotate to make editing easy.  Once the script is added you should have a new folder created at /mnt/user/mount_unionfs.  Inside this folder create your media folders i.e. /mnt/user/mount_unionfs/google_vfs/movies and /mnt/user/mount_unionfs/google_vfs/tv_shows.  These are the folders to add to plex, radarr, sonarr etc.

How it works is new files are written to the local RW part of the union mount (/mnt/user/rclone_upload), but the dockers can't distinguish between whether the file is local or in the cloud because they are checking /mnt/user/mount_unionfs. 

A later script moves files from /mnt/user/rclone_upload to the cloud; to dockers the files are still in /mnt/user/mount_unionfs, so nothing has changed for them.

Update: delete the teamdrive section if you don't need it

<b>4. Rclone upload script - see https://github.com/BinsonBuzz/unraid_rclone_mount for latest script</b>

I run this every hour to move files from my local drive /mnt/user/rclone_upload to the cloud.  I have set --bwlimit at 9500K as I find that even that even though this theoretically means I transfer more than google's 750GB/day limit, lower limits don't get me up to this mark.  Experiment with your setup if you've got enough upstream to upload 750GB/day.

I've also added --min age 30mins to again stop any premature uploads.

The script includes some exclusions to stop partial files etc getting uploaded.

Update: I have a 2nd upload script on github to move files to the teamdrive for an extra 750GB/day quota.  My script below only runs against user0 as I use my second script to upload from my cache.  If you using one script, just change user0 to user

<b>5. Unionfs cleanup script - see https://github.com/BinsonBuzz/unraid_rclone_mount for latest script</b>

The 'problem' with unionfs is that when it needs to delete a file from the cloud e.g. you have a better quality version of a file, it doesn't actually delete it - it 'hides' it from the mount so it appears deleted, but the file actually still exists.  So, if you in the future create a new mount or access the cloud drive via another means, the files will still be there potentially creating a very messy library.

This script cleans up the cloud files and actually deletes them - I run this a few times a day.


Update: The script now cleans the teamdrive.  Delete the two lines with newPath2 if you don't use a teamdrive

<b>6. Unmount Script - - see https://github.com/BinsonBuzz/unraid_rclone_mount for latest script</b>

I use this at array start to make sure all the 'check' files have been removed properly in case of an unclean shutdown, to ensure the next mount goes smoothly.  

Update: delete teamdrive fusermount line if don't need
