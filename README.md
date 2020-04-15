Rclone Mount & Upload Scripts for Plex Users
Collection of scripts to create rclone google mounts to allow fast launch times with Plex (or Emby).

The main thread for more support: https://forums.unraid.net/topic/75436-guide-how-to-use-rclone-to-mount-cloud-drives-and-play-files/.

Credits:

Thanks to SenPaiBox and the Unraid community for help in refining the scripts.

Unraid Users Requirements:

Unraid Rclone Plugin
1.5.1 or higher needed Details
Installs rclone and allows the creation of remotes and mounts
Optional: Unraid CA User Scripts Plugin
Best way to run scripts Details
Optional: Create Service Accounts (follow steps 1-4).
To mass rename the service accounts use the following steps:
Place Auto-Genortated Service Accounts into /mnt/user/appdata/other/rclone/service_accounts/
Run the following in terminal/ssh
Move to directory: cd /mnt/user/appdata/other/rclone/service_accounts/
Dry Run:
n=1; for f in *.json; do echo mv "$f" "sa_gdrive_upload$((n++)).json"; done
Mass Rename:
n=1; for f in *.json; do mv "$f" "sa_gdrive_upload$((n++)).json"; done
Non-unRaid Users
Other users need to install rclone and use their preferred way to schedule cron jobs. The scripts install mergerfs, which I think should work for other systems.

Optional: Create Service Accounts (follow steps 1-4).
For mass renaming: See steps in Unraid Requirements above
How It Works

Rclone is used to access files on your google drive and to mount them in a folder on your server e.g. mount a gdrive remote called gdrive_vfs: at /mnt/user/mount_rclone/gdrive_vfs
Mergerfs is used to merge files from your rclone mount (/mnt/user/mount_rclone/gdrive_vfs) with local files that exist on your server and haven't been uploaded yet (e.g. /mnt/user/local/gdrive_vfs) in a new mount /mnt/user/mount_unionfs/gdrive_vfs
This mergerfs mount allows files to be played by dockers such as Plex, or added to by dockers like radarr etc without the dockers even being aware that some files are local and some are remote. It just doesn't matter
The use of a rclone vfs remote allows fast playback, with files streaming within seconds
New files added to the mergerfs share are actually written to the local share, where they will stay until the upload script processes them
An upload script is used to upload files in the background from the local folder to the remote. This activity is masked by mergerfs i.e. to plex, radarr etc files haven't 'moved'
Getting Started
Rclone remote setup
Install the rclone plugin and via command line run rclone config and create 1-2 remotes:
Required: gdrive: - a drive remote that connects to your gdrive account. Recommend creating your own client_id
Recommended: gdrive_media_vfs: - a crypt remote that is mounted locally and decrypts the encrypted files uploaded to gdrive:
Once complete your rclone_config file should look something like this:
[gdrive]
type = drive
client_id = UNIQUE CLIENT_ID
client_secret = MATCHING_UNIQUE_SECRET
scope = drive
root_folder_id = xxxx
token = {"xxxxx"}
server_side_across_configs = true

[gdrive_media_vfs]
type = crypt
remote = gdrive:crypt
filename_encryption = standard
directory_name_encryption = true
password = PASSWORD1
password2 = PASSWORD2

Or, like this if using service accounts:

[gdrive]
type = drive
scope = drive
service_account_file = /mnt/user/appdata/other/rclone/service_accounts/sa_gdrive.json
team_drive = TEAM DRIVE ID
server_side_across_configs = true

[gdrive_media_vfs]
type = crypt
remote = gdrive:crypt
filename_encryption = standard
directory_name_encryption = true
password = PASSWORD1
password2 = PASSWORD2

If you need help doing this, please consult the forum thread above.

It is advisable to create your own client_id to avoid API bans. More Details
Mount script
Create a new script using the the user scripts plugin and paste in the rclone_mount script
Edit the config lines at the start of the script to choose your remote name, paths etc
Choose a suitable cron job. I run this script on a 10 min */10 * * * * schedule so that it automatically remounts if there’s a problem.
The script:
Checks if an instance is already running, remounts (if cron job set) automatically if mount drops
Mounts your rclone gdrive remote
Installs mergerfs and creates a mergerfs mount
Starts dockers that need the mergerfs mount e.g. plex, radarr
Upload script
Create a new script using the the user scripts plugin and paste in the rclone_mount script
Edit the config lines at the start of the script to choose your remote name, paths etc - USE THE SAME PATHS
Choose a suitable cron job e.g hourly
Features:
Checks if rclone is installed correctly
sets bwlimits
There is a cap on uploads by google of 750GB/day. I have added bandwidth scheduling to the script so you can e.g. set an overnight job to upload the daily quota at 30MB/s, have it trickle up over the day at a constant 10MB/s, or set variable speeds over the day
The script now stops once the 750GB/day limit is hit (rclone 1.5.1+ required) so there is more flexibility over upload strategies
I've also added --min age 10mins to stop any premature uploads and exclusions to stop partial files etc getting uploaded.
Cleanup script
Create a new script using the the user scripts plugin and set to run at array start (recommended) or array stop
Using Mergerfs

Once the scripts are added you should have a new folder created at /mnt/user/mount_mergerfs/name_of_remote. 

Inside this folder, add files that you want uploading to google i.e. create your media folders here
This includes any files downloaded by e.g. nzbget (i.e use /mnt/user/mount_unionfs/downloads) that you want radarr etc to move e.g. /mnt/user/mount_mergerfs/google_vfs/movies/Star Wars and /mnt/user/mount_mergerfs/google_vfs/Peppa Pig.
To get the best performance out of mergerfs, map dockers to /user --> /mnt/user
Then within the docker webui navigate to the relevant folder within the mergerfs share e.g. /user/mount_unionfs/downloads or /user/mount_unionfs/movies. These are the folders to map to plex, radarr, sonarr,nzbget etc
DO NOT MAP any folders from local or the rclone mount
DO NOT create mappings like /downloads or /media for your dockers. Only use /user --> /mnt/user if you want to ensure the best performance from mergerfs when moving and editing files within the mount
Troubleshooting
If you need to unmount manually the command to use is:

fusermount -uz /path/to/remote