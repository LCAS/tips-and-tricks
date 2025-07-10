# OneDrive for Linux 

I love microsoft so I was estatic when I realised I needed a new tool to be approved to allow me to more smoothly continue with my work. 
I had the pleasure of liasing with the uni central IT team to justify why this tool would be helpful for computer scientists and researchers, and have been able to get the Onedrive for linux tool approved. So it is now possible to sync up your onedrive and sharepoint directories directly to your Linux Machine. 

As a caveat, this means that YOU MUST make sure the storage of the data is secure and compliant to GDPR or even stricter security rules if any projects require that!
 
Onedrive itself doesn't provide this, once the data is on the machine, you won't be able to remove it or encrypt it suddenly remotely though some toggle in your Microsoft account. Make sure the drive you use to store this data is secure and encypted.   

This has been tested on `PopOS 22.04` and `Arch Linux`, both work well. 

- Repo: https://github.com/abraunegg/onedrive/
- Usage Guide for personal account: https://github.com/abraunegg/onedrive/blob/master/docs/usage.md
- Usage Guide for a Sharepoint: https://github.com/abraunegg/onedrive/blob/master/docs/sharepoint-libraries.md

I have had a scare where I deleted all of my files (sharepoint specifically) during confoguration of the sharpoint directory. Luckly this was cause I changes the directory it pointed to which meant my original local directory was untouched.
It basically thought that I decided I wanted to delete all of the files in my onedrive. 

**So**
  1. Make sure you confidently know where you want your files to sync from/to before starting.
  2. If you decide to change directories then DELETE your sharepoint confguration files and start from scratch as if you were configuring a new sharepoint

It's simple if you follow the guide, but room for error can be quite consequential if you have a constant sync/monitoring process in the background because moving things around just means that it thinks everything has been deleted.


--- 
Note: you can probably avoid the issue and let yourself move things around as you please if you use a Symlink, but I will not advise any more than that, up to you to try and find out if there are consequences.

---

I have yet to test out the following enhancements, but I actually need to get on with my PhD so this is for another week:
- A GUI for configuration management: [OneDrive Client for Linux GUI](https://github.com/bpozdena/OneDriveGUI) 
- Colorful log output terminal modification: [OneDrive Client for Linux Colorful log Output](https://github.com/zzzdeb/dotfiles/blob/master/scripts/tools/onedrive_log)
- System Tray Icon: [OneDrive Client for Linux System Tray Icon](https://github.com/DanielBorgesOliveira/onedrive_tray)
