# unraid_rclone_mount

unRAID scripts to create rclone vfs mounts on unraid to allow fast launch times with Plex (or Emby). 

The main thread for more support:

https://forums.unraid.net/topic/75436-guide-how-to-use-rclone-to-mount-cloud-drives-and-play-files/

<b>Plugins needed</b>

<li>Rclone beta – installs rclone and allows the creation of remotes and mounts</li>
<li>User Scripts – to run scripts</li>
<br/>
<b>How It Works </b>
<br/>
<li>Rclone is used to access files on your google drive in a mount </li>
<li>Mergerfs is used to merge files on your rclone/google mount with local files that haven't been uploaded yet in another  mount </li>
<li>Dockers that need to play files (Plex, Emby) and dockers that need to add new files (Sonarr, Radarr, nzbget, transmission etc) <b>ALL</b> are mapped to folders <b>WITHIN</b> the <b>MERGERFS MOUNT </b>, not the real local location or the rclone mount </li>
<br/>
<
<b>1.       Rclone remote setup </b> 

Install the rclone plugin and via command line run rclone config and create 2 remotes: 

<li>gdrive: - a drive remote that connects to your gdrive account.  Recommend creating your own client_id</li>
<li>gdrive_media_vfs: - a crypt remote that is mounted locally and decrypts the encrypted files uploaded to gdrive:</li>

Once complete your rclone_config file should look something like this:
<br/>
[gdrive]
<br/>type = drive
<br/>client_id = xxxx.apps.googleusercontent.com
<br/>client_secret = xxxxx
<br/>scope = drive
<br/>root_folder_id = xxxx
<br/>service_account_file = 
<br/>token = {"xxxxx"}

[gdrive_media_vfs]
<br/>type = crypt
<br/>remote = gdrive:crypt
<br/>filename_encryption = standard
<br/>directory_name_encryption = true
<br/>password = xxxx
<br/>password2 = -xxxxx
<br/>
If you need help doing this, please consult the forum thread above.  

<b>2.       Create Mountcheck files</b>

This blank file is used in the following scripts to verify if the mounts have been created properly.  Run these commands:
<br>
<i>touch mountcheck</i>
<br>
<i>rclone copy mountcheck gdrive_media_vfs: -vv --no-traverse</i>
<br>
<i>rclone copy mountcheck tdrive_media_vfs: -vv --no-traverse</i>

<b>3.      Mount script</b>

Create a new script in user scripts to create the rclone mount, unionfs mount and start dockers that need the mounts.  I run this script on a 10 min */10 * * * * schedule so that it automatically remounts if there’s a problem. 

The script:

<li>Checks if an instance is already running</li>
<li>Update: Mounts rclone gdrive and tdrive remotes</li>
<li>Update: Mounts unionfs creating a 3-way union between rclone gdrive remote, tdrive remote and local files stored in /mnt/user/rclone_upload</li>
<li>Starts dockers that need the unionfs mount e.g. radarr</li>
<li>New: used rclone rc to populate the directory cache</li>
<br>
I've tried to annotate to make editing easy.  Once the script is added you should have a new folder created at /mnt/user/mount_unionfs.  Inside this folder create your media folders i.e. /mnt/user/mount_unionfs/google_vfs/movies and /mnt/user/mount_unionfs/google_vfs/tv_shows.  These are the folders to add to plex, radarr, sonarr etc.

How it works is new files are written to the local RW part of the union mount (/mnt/user/rclone_upload), but the dockers can't distinguish between whether the file is local or in the cloud because they are checking /mnt/user/mount_unionfs. 

A later script moves files from /mnt/user/rclone_upload to the cloud; to dockers the files are still in /mnt/user/mount_unionfs, so nothing has changed for them.

<b>4. Rclone upload script</b>

I run this every hour to move files from my local drive /mnt/user/rclone_upload to the cloud.  I have set --bwlimit at 9500K as I find that even that even though this theoretically means I transfer more than google's 750GB/day limit, lower limits don't get me up to this mark.  Experiment with your setup if you've got enough upstream to upload 750GB/day.

I've also added --min age 30mins to again stop any premature uploads.

The script includes some exclusions to stop partial files etc getting uploaded.

Update: I have a 2nd upload script on github to move files to the teamdrive for an extra 750GB/day quota.  My script below only runs against user0 as I use my second script to upload from my cache.  If you using one script, just change user0 to user

<b>5. Unionfs cleanup script</b>

The 'problem' with unionfs is that when it needs to delete a file from the cloud e.g. you have a better quality version of a file, it doesn't actually delete it - it 'hides' it from the mount so it appears deleted, but the file actually still exists.  So, if you in the future create a new mount or access the cloud drive via another means, the files will still be there potentially creating a very messy library.

This script cleans up the cloud files and actually deletes them - I run this a few times a day.

<b>6. Unmount Script</b>

I use this at array start to make sure all the 'check' files have been removed properly in case of an unclean shutdown, to ensure the next mount goes smoothly.  
