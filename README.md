# Unraid Rclone Scripts

Unraid scripts to create rclone vfs mounts on unraid to allow fast launch times with Plex (or Emby).  They should work on other systems, just take care with your paths.

The main thread for more support:

https://forums.unraid.net/topic/75436-guide-how-to-use-rclone-to-mount-cloud-drives-and-play-files/

<b>Plugins Needed</b>
<ul>
  <li><b>Unraid Rclone Plugin</b>
<ul>
<li>1.5.1 or higher needed <b><a href="https://forums.unraid.net/topic/51633-plugin-rclone/">Details</a></b></li>
  <li>Installs rclone and allows the creation of remotes and mounts</li>
 </ul>
  <li><b>Unraid CA User Scripts Plugin</b></li>
  <ul>
<li>Best way to run scripts<b><a href="https://forums.unraid.net/topic/48286-plugin-ca-user-scripts/">Details</a></b>
  </ul>
</ul>
<br/>
<b>How It Works </b>
<br/><br>
<ul>
<li>Rclone is used to access files on your google drive and to mount them in a folder on your server e.g. mount a gdrive remote called gdrive_vfs: at /mnt/user/mount_rlone/gdrive_vfs </li>
<li>Mergerfs is used to merge files from your rclone mount (/mnt/user/mount_rlone/gdrive_vfs) with local files that exist on your server and haven't been uploaded yet (e.g. /mnt/user/local/gdrive_vfs) in a new mount /mnt/user/mount_unionfs/gdrive_vfs
  <ul>
    <li>This mergerfs mount allows files to be played by dockers such as Plex, or added to by dockers like radarr etc without the dockers even being aware that some files are local and some are remote.  It just doesn't matter</li>
    <li> the use of a rclone vfs remote allows fast playback, with files streaming within seconds</li>
  </ul>
<li>An upload script is used to upload files in the background from the local folder to the remote.  This activity is masked by mergerfs i.e. to plex, radarr etc files haven't 'moved'</li>
</ul>
<li>Dockers that need to play files (Plex, Emby) and dockers that need to add new files (Sonarr, Radarr, nzbget, transmission etc) <b>ALL</b> are mapped to folders <b>WITHIN</b> the <b>MERGERFS MOUNT </b>, not the real local location or the rclone mount </li>


<br/>
<b>1.       Rclone remote setup </b> 
<br><br>
Install the rclone plugin and via command line run rclone config and create 2 remotes: 
<br>
<li>gdrive: - a drive remote that connects to your gdrive account.  Recommend creating your own client_id</li>
<li>gdrive_media_vfs: - a crypt remote that is mounted locally and decrypts the encrypted files uploaded to gdrive:</li>
<br/>
Once complete your rclone_config file should look something like this:
<br/>
<br/>
[gdrive]
<br/>type = drive
<br/>client_id = xxxx.apps.googleusercontent.com
<br/>client_secret = xxxxx
<br/>scope = drive
<br/>root_folder_id = xxxx
<br/>service_account_file = 
<br/>token = {"xxxxx"}
<br/><br/>
[gdrive_media_vfs]
<br/>type = crypt
<br/>remote = gdrive:crypt
<br/>filename_encryption = standard
<br/>directory_name_encryption = true
<br/>password = xxxx
<br/>password2 = -xxxxx
<br/><br/>
If you need help doing this, please consult the forum thread above.  
<br/><br/>
<b>2.       Create Mountcheck files</b>
<br><br>
This blank file is used in the following scripts to verify if the mounts have been created properly.  Run these commands:
<br>
<i>touch mountcheck</i>
<br>
<i>rclone copy mountcheck gdrive_media_vfs: -vv --no-traverse</i>
<br><br>
<b>3.      Mount script</b>
<br><br>
Create a new script in user scripts to create the rclone mount, mergerfs mount and start dockers that need the mounts.  I run this script on a 10 min */10 * * * * schedule so that it automatically remounts if there’s a problem. 
<br>
The script:
<br>
<li>Checks if an instance is already running</li>
<li>Update: Mounts rclone gdrive remote</li>
<li>Update: Mounts mergerfs creating a 3-way union between rclone gdrive remote and local files stored in /mnt/user/local/google_vfs</li>
<li>Starts dockers that need the mergerfs mount e.g. radarr</li>
<br>
I've tried to annotate to make editing easy.  Once the script is added you should have a new folder created at /mnt/user/mount_mergerfs/google_vfs.  Inside this folder add files that you want uploading to google create your media folders i.e. /mnt/user/mount_mergerfs/google_vfs/movies/Star Wars and /mnt/user/mount_mergerfs/google_vfs/Peppa Pig.  These are the folders to map to plex, radarr, sonarr,nzbget etc - DO NOT MAP any folders from local or the rclone mount.
<br><br>
How it works is new files are written in the background to the local RW part of the mergerfs mount (/mnt/user/local/google_vfs), but the dockers can't distinguish between whether the file is local or in the cloud because they are checking /mnt/user/mount_mergerfs. 
<br><br>
A later script moves new files added to /mnt/user/local/google_vfs to the cloud (excluding partial files and files in the download folder so that files can seed etc.); to dockers the files are still in /mnt/user/mount_mergerfs and haven't moved, so nothing has changed for them.
<br><br>
<b>4. Rclone upload script</b>
<br><br>
I run this every hour to move new files from my local drive /mnt/user/local/google_vfs to the cloud.  I have set --bwlimit at 9500K as I find that even that even though this theoretically means I transfer more than google's 750GB/day limit, lower limits don't get me up to this mark.  Experiment with your setup if you've got enough upstream to upload 750GB/day.
<br><br>
I've also added --min age 30mins to again stop any premature uploads.
<br><br>
The script includes some exclusions to stop partial files etc getting uploaded.
<br><br>
<b>5. Unmount Script</b>
<br><br>
I use this at array start to make sure all the 'check' files have been removed properly in case of an unclean shutdown, to ensure the next mount goes smoothly.  
