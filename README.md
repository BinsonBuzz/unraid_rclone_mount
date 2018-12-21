# unraid_rclone_mount
scripts to create rclone vfs mounts on unraid to allow fast launch times with Plex (or Emby).

The main thread

https://forums.unraid.net/topic/75436-guide-how-to-use-rclone-to-mount-cloud-drives-and-play-files/

<b>Plugins needed</b>

Rclone beta – installs rclone and allows the creation of remotes and mounts
User Scripts – controls how mounts get created


<b>Optional Plugins</b>

Nerd Tools - used to install Unionfs which allows a 2nd mount to be created that merges the rclone mount with files locally e.g. new TV episodes that haven’t been uploaded yet, so that dockers like sonar, radar etc can see that you’ve already got the files and don’t try to add them to your library again.  In the future hopefully this will be replaced with rclone’s new Union allowing for an all-in-one solution

