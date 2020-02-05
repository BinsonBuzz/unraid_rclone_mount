# Unraid Rclone Scripts

Collection of Unraid scripts to create rclone vfs mounts on unraid to allow fast launch times with Plex (or Emby).  They should work on other systems, just take care with your paths.

The main thread for more support:

https://forums.unraid.net/topic/75436-guide-how-to-use-rclone-to-mount-cloud-drives-and-play-files/

Credits:

Thanks to <a href="https://github.com/SenpaiBox">SenPaiBox</a> and the Unraid community for help in refining the scripts.

<b>Plugins Needed</b>
<ul>
	<li><b>Unraid Rclone Plugin</b>
		<ul>
			<li>1.5.1 or higher needed <b><a href="https://forums.unraid.net/topic/51633-plugin-rclone/">Details</a></b></li>
			<li>Installs rclone and allows the creation of remotes and mounts</li>
		</ul>
	<li><b>Unraid CA User Scripts Plugin</b></li>
		<ul>
			<li>Best way to run scripts<b> <a href="https://forums.unraid.net/topic/48286-plugin-ca-user-scripts/">Details</a></b>
		</ul>
</ul>
<b>How It Works </b>
<ol>
	<li>Rclone is used to access files on your google drive and to mount them in a folder on your server e.g. mount a gdrive remote called gdrive_vfs: at /mnt/user/mount_rlone/gdrive_vfs </li>
	<li>Mergerfs is used to merge files from your rclone mount (/mnt/user/mount_rlone/gdrive_vfs) with local files that exist on your server and haven't been uploaded yet (e.g. /mnt/user/local/gdrive_vfs) in a new mount /mnt/user/mount_unionfs/gdrive_vfs</li>
		<ul>
			<li>This mergerfs mount allows files to be played by dockers such as Plex, or added to by dockers like radarr etc without the dockers even being aware that some files are local and some are remote.  It just doesn't matter</li>
			<li>The use of a rclone vfs remote allows fast playback, with files streaming within seconds</li>
			<li>New files added to the mergerfs share are actually written to the local share, where they will stay until the upload script processes them
		</ul>
	<li>An upload script is used to upload files in the background from the local folder to the remote.  This activity is masked by mergerfs i.e. to plex, radarr etc files haven't 'moved'</li>
</ol>
<b>Getting Started </b>
<ol>
	<li>Rclone remote setup </li> 
		<ul>
			<li>Install the rclone plugin and via command line run rclone config and create 2 remotes:</li> 
				<ol>
					<li>gdrive: - a drive remote that connects to your gdrive account.  Recommend creating your own client_id</li>
					<li>gdrive_media_vfs: - a crypt remote that is mounted locally and decrypts the encrypted files uploaded to gdrive:</li>
				</ol>
		</ul>
	<p/>
	<li>Once complete your rclone_config file should look something like this:</li>
	<p/><p/><i>
[gdrive]
<br/>type = drive
<br/>client_id = UNIQUE CLIENT_ID
<br/>client_secret = MATCHING_UNIQUE_SECRET
<br/>scope = drive
<br/>root_folder_id = xxxx
<br/>service_account_file = 
<br/>token = {"xxxxx"}
<br/>server_side_across_configs = true
<p/>
[gdrive_media_vfs]
<br/>type = crypt
<br/>remote = gdrive:crypt
<br/>filename_encryption = standard
<br/>directory_name_encryption = true
<br/>password = PASSWORD1
<br/>password2 = PASSWORD2
	</i></p/>
If you need help doing this, please consult the forum thread above.
</p>
<b> It is advisable to create your own client_id to avoid API bans.  <a href="https://rclone.org/drive/#making-your-own-client-id">More Details</a></b>
<p/>
	<li>Mount script</li>
		<ul>
			<li>Create a new script using the the user scripts plugin and paste in the rclone_mount script</li>
			<li>Edit the config lines at the start of the script to choose your remote name, paths etc</li>
			<li>Choose a suitable cron job. I run this script on a 10 min */10 * * * * schedule so that it automatically remounts if there’s a problem.</li>
			<li>The script:</li>
			<ul>
				<li>Checks if an instance is already running, remounts (if cron job set) automatically if mount drops</li>
				<li>Mounts your rclone gdrive remote</li>
				<li>Installs mergerfs and creates a mergerfs mount</li>
				<li>Starts dockers that need the mergerfs mount e.g. plex, radarr</li>
			</ul>
		</ul>
	</p/>
	<li>Upload script</li>
		<ul>
			<li>Create a new script using the the user scripts plugin and paste in the rclone_mount script</li>
			<li>Edit the config lines at the start of the script to choose your remote name, paths etc - USE THE SAME PATHS</li>
			<li>Choose a suitable cron job e.g hourly</li>
			<li>Features:</li>
			<ul>
				<li>Checks if rclone is installed correctly</li>
				<li>sets bwlimits</li>
				<li>There is a cap on uploads by google of 750GB/day.  I have added bandwidth scheduling to the script so you can e.g. set an overnight job to upload the daily quota at 30MB/s, have it trickle up over the day at a constant 10MB/s, or set variable speeds over the day</li>
				<li>The script now stops once the 750GB/day limit is hit (rclone 1.5.1+ required) so there is more flexibility over upload strategies</li>
				<li>I've also added <i>--min age 10mins</i> to stop any premature uploads and exclusions to stop partial files etc getting uploaded.</li>
			</ul>
		</ul>
	</p/>
	<li>Cleanup script</li>
		<ul>
			<li>Create a new script using the the user scripts plugin and set to run at array start (recommended) or array stop</li>
		</ul>
</ol>
<p/>
<b>Using Mergerfs</b>
<p/>
Once the scripts are added you should have a new folder created at /mnt/user/mount_mergerfs/name_of_remote. 
<ul>
<li>Inside this folder, add files that you want uploading to google i.e. create your media folders here</li>
<li>This includes any files downloaded by e.g. nzbget (i.e use /mnt/user/mount_unionfs/downloads) that you want radarr etc to move e.g. /mnt/user/mount_mergerfs/google_vfs/movies/Star Wars and /mnt/user/mount_mergerfs/google_vfs/Peppa Pig.</li>
<li>To get the best performance out of mergerfs, map dockers to /user   --> /mnt/user </li>
<ul>
<li>Then within the docker webui navigate to the relevant folder within the mergerfs share e.g. /user/mount_unionfs/downloads or /user/mount_unionfs/movies. These are the folders to map to plex, radarr, sonarr,nzbget etc</li>
<li><b>DO NOT MAP any folders from local or the rclone mount</b></li>
<li><b>DO NOT create mappings like /downloads or /media for your dockers.  Only use /user --> /mnt/user if you want to ensure the best performance from mergerfs when moving and editing files within the mount</b></li>
</ul>
</ul>
