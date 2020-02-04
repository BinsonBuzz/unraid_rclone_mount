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
	Once complete your rclone_config file should look something like this:
	<p/>
[gdrive]
<br/>type = drive
<br/>client_id = xxxx.apps.googleusercontent.com
<br/>client_secret = xxxxx
<br/>scope = drive
<br/>root_folder_id = xxxx
<br/>service_account_file = 
<br/>token = {"xxxxx"}
<p/>
[gdrive_media_vfs]
<br/>type = crypt
<br/>remote = gdrive:crypt
<br/>filename_encryption = standard
<br/>directory_name_encryption = true
<br/>password = xxxx
<br/>password2 = -xxxxx
</p/>
If you need help doing this, please consult the forum thread above.  
</ol>
