---
title: Setting up authentication using PAM_U2F and Yubikeys in Bazzite Linux
description: "How I set up up Sudo, Lock Screen and Login authentication via a Yubikey using Pam_u2F and FIDO2"
date: 2026-06-02T21:30:11.453Z
preview: ""
draft: false
tags: [Yubikey, PAM, Bazzite, sudo, login]
categories: []
showComments: true
---

**Introduction**

My original aim for this project was to enable a Windows Hello-style experience for unlocking or logging into my Bazzite desktop. This would be achieved using either a USB fingerprint reader or a webcam with a built-in IR camera for Windows Hello.
Unfortunately, the USB fingerprint reader I own, based on the Focal Systems FT9201, has limited ability to work with Bazzite — or Linux-based operating systems in general — due to a lack of proper, stable driver support. There are a couple of projects that attempt to provide a working driver (https://github.com/banianitc/ft9201-fingerprint-driver/ and https://github.com/ryenyuku/libfprint-ft9201), but both had numerous show-stopping issues for me personally.
Another blow: Howdy, the facial recognition program for Linux-based operating systems, would need to be installed via a Fedora third-party COPR repository (COPR is a Fedora community-built package hosting service). This would need to be layered onto Bazzite, but because Bazzite is mostly immutable (meaning the system is provided as a read-only image that is updated as a whole), layered third-party packages can occasionally be broken by system updates.
That didn't sound like the maintenance-free, easy solution I was looking for. It is probably possible to get it working, and hopefully someday a more officially supported installation method will become available.

**The Solution — YubiKey**
Thankfully, I had another solution: the YubiKey.
This is a hardware authentication device, a USB-stick-sized gadget that can be registered with a service such as Google or Bitwarden. During registration, the key generates a unique public/private key pair for that service. It sends the public half to the service to keep on file, while the private half remains locked inside the key and never leaves it (nor can it be extracted).
When you next log in to a service with a YubiKey registered — Google, for example — you can use it as your second factor after entering your password. The browser sends a cryptographic challenge to the key over USB, you touch the gold contact to give consent (this can be disabled, but I don't recommend it as it as otherwise malware could easily use the key to authenticate, reducing security), and the key signs the challenge with its private key before sending the signed response back.
Modern standards such as FIDO2 also support optional user verification through a PIN or a fingerprint on devices like the YubiKey Bio. This means that even if your key were stolen, it could not be used without that additional factor. After too many failed attempts, the FIDO application may become blocked and require recovery or reset procedures, so choose your PIN carefully and keep it safe.
For my use case, the main advantage over the USB fingerprint reader or Howdy is that the YubiKey has native support on Bazzite through a tool called pam-u2f, which is available in the standard Fedora repositories. Installation is therefore straightforward. There's no fighting with Bazzite's immutable design, and edits to PAM configuration files such as /etc/pam.d/sudo are persistent, so updates to Bazzite should not break this setup.
YubiKey also provides a Linux command-line management utility called ykman.

**YubiKey Pre-Setup and Things to Bear in Mind**
Even though a YubiKey configured for login or sudo will still allow password fallback (provided you configure PAM appropriately), I strongly recommend purchasing a second key as a backup. I also recommend maintaining a strong password and storing a copy in a secure location such as a safe.
This ensures that you should never be permanently locked out of your machine.
The other thing I would recommend is setting a fixed hostname. FIDO2 credentials are associated with an origin, and in the case of pam-u2f that origin is derived from the hostname. If the hostname changes, authentication can break.
Because Bazzite did not automatically maintain a fixed hostname on my installation, various events could override it and break YubiKey authentication for login, sudo, and other PAM-protected services.
I encountered this issue after dual-booting into Windows. My router assigned my Windows hostname to the IP address, and Bazzite subsequently adopted that hostname when I booted back into Linux. This caused my pam_u2f configuration to stop working until I restored the original hostname.I used 
sudo hostnamectl set-hostname your-name
to set a fixed hostname.
If you use a Chromium-based browser (I tested Edge, Chrome, and Chromium), I do not recommend replacing your primary graphical login password with YubiKey authentication. These browsers use KDE Wallet to store encrypted cookies and credentials. KDE Wallet is normally unlocked using your login password; if you bypass password authentication entirely, KDE Wallet may remain locked.
In my testing, this resulted in browsers entering an infinite page-loading loop unless Incognito Mode was used.
Firefox uses its own encrypted storage mechanism and was unaffected.
One final point: YubiKey firmware cannot be upgraded. This is an intentional security design decision. Don't purchase an older model expecting to update it later. For the setup described in this article, you'll want a YubiKey with firmware version 5.2.3 or newer.


**How I Set Up My YubiKeys**

**Initial Setup**

First, you'll need to install ykman (Yubico's key-management tool):
which ykman || sudo rpm-ostree install yubikey-manager
Plug in your key and confirm it's recognised:
ykman info
Then change the PIN on your key, for the cases where FIDO requires user verification:
ykman fido access change-pin
Next, install pam-u2f:
sudo rpm-ostree install pam-u2f
Reboot, then verify it landed:
ls /usr/lib64/security/pam_u2f.so
You'll also want libfido2 (pre-installed in my case — check with rpm -q libfido2 and install it with sudo rpm-ostree install libfido2 if it isn't there).
pam-u2f (Pluggable Authentication Module — Universal 2nd Factor) is the module that allows authentication via a USB key like the YubiKey; libfido2 is the underlying library that implements the FIDO2 protocol.
After installation, your key (or keys) need to be enrolled with pam-u2f:
mkdir -p ~/.config/Yubico
pamu2fcfg > ~/.config/Yubico/u2f_keys
If you have two keys, run the following for the second key:
pamu2fcfg -n >> ~/.config/Yubico/u2f_keys
This appends the second key to the same line as the first, so pam-u2f treats them as alternatives. Verify with:
cat ~/.config/Yubico/u2f_keys
It should look like your username, a colon, a long string of numbers and letters, another colon, then another long string. The long strings are the key credentials that PAM will use to authenticate your keys.

A quick note, if you wanted user verification, for example a pin code or for a biometric yubikey, a fingerprint, then adding –v to pamu2fcfg > ~/.config/Yubico/u2f_keys, would add this as a requirement. I did not do this as these keys will never leave my house, so the added convenience is fine. When I add keys to my laptop however, I will be using user verification however. 
Step 3: SAFETY
Whenever you're messing around with authentication files, open a second terminal and run sudo -i (the -i ensures that the sudo privileges won't expire until the terminal is closed). This is in case something gets screwed up and you can no longer authenticate as root; the open terminal allows you to revert your changes.
Before editing any files, please check that your TTY interface works, this is an additional fallback. Press ctrl + alt + f3, this load TTY mode, type in your user and password to check if they work, then press ctrl +alt + f2 to go back. This will allow you to revert any changes made if something breaks. 
Allowing the YubiKey to authenticate
I started with sudo, as that would most easily allow me to test if this was working. I ran:
sudo nano /etc/pam.d/sudo
This brings up a text file with rows that look like auth include system-auth and account include system-auth.
As the first line I added:
auth sufficient pam_u2f.so cue
This contains all the instructions needed. The YubiKey is at the top so it will be tried first; auth means the module is being used for authentication; sufficient means if this module suceeds, grant access, if it fails, ignore and move onto the next line, pam_u2f.so is the module doing the authentication; cue tells it to prompt you to touch the key so you know it's waiting.
Ctrl+O to save, Enter to confirm, then Ctrl+X to exit. To test if it worked, run:
sudo -k
sudo whoami
It should ask you to touch the key to authenticate, then reply with root. Repeat with the second key if you have one. Then test again with only your password to confirm that the sufficient line passes through to the next.

In my case it was successful, so I proceeded to /etc/pam.d/kde (for the lock screen) and /etc/pam.d/kscreensaver (mostly for any legacy programs that reference this file). Same method as above:
sudo nano /etc/pam.d/kde
Add auth sufficient pam_u2f.so cue as the top line, Ctrl+O, Enter, Ctrl+X, then test both keys and password by locking your screen.
For polkit, in my case the configuration file lived under /usr/lib/pam.d/polkit-1, which would not persist between updates. Thankfully, copying the file to /etc/pam.d/polkit-1 will make it persistent:
sudo cp /usr/lib/pam.d/polkit-1 /etc/pam.d/polkit-1
sudo nano /etc/pam.d/polkit-1
Add auth sufficient pam_u2f.so cue at the top. Test with:
pkexec ls /root
This should produce a dialogue box, your key should start to flash, then when pressed it should list the root directory contents. 
If you wish to make the Yubikey a requirement for authentication, then you could change sufficient to required. With required, even if a later password check succeeds, the overall authentication still fails — so losing the key really does lock you out, and recovery means dropping to a TTY and editing the modified files to remove the pam_u2f line. I don’t recommend this as it means to if you lose your key, then authenticating back in would require editing the modified files using TTY to remove the pam_u2f line. 

Logging in with the YubiKey

My version of Bazzite was using plasmalogin for login management, which is a newer system than SDDM. plasmalogin has three files used for authentication: plasmalogin, plasmalogin-greeter, and plasmalogin-autologin. I edited plasmalogin. k. To make restoring easier, I copied my plasmalogin file
sudo cp /usr/lib/pam.d/plasmalogin /etc/pam.d/plasmalogin 
The /etc/ version of the file is preferred by Bazzite, and would be persistent, so I edited that file but, if deleted the system would resort back to /usr/lib/pam.d/plasmalogin. Then
sudo nano /etc/pam.d/plasmalogin
to edit the file, however there is a catch with this file, the 1st line is
auth [success=done ignore=ignore default=bad] pam_selinux_permit.so
This is related to SELinux (Security-Enhanced Linux, a kernel security module that handles mandatory access control), which I left alone — too much risk of breaking something. Add auth sufficient pam_u2f.so cue as the second line, immediately after the SELinux one. This is where KDEwallet partially breaks, because sufficient passes as soon as the check is complete, then the line line to authenticate KDE wallet never runs. 
After that, logout, select user and in my case it won’t try and use the Yubikey straight away, instead it shows the password field, this is skippable by just pressing enter, then it activates the yubikey. Test your backup key if you have one, then password. If the login fails, press ctrl + alt + f3, then run 
sudo rm /etc/pam.d/plasmalogin
To delete the edited file.

Conclusion

Other than the the dual-boot hostname incident and the kdewallet issues mentioned above, and the login screen loading password field first, this setup has been flawless. It has allowed me to set a much more secure root password, as I don;t have to type it every single time I want to authenticate. I have also added my Yubikeys as a 2nd factor to a number of online services I use, such as Bitwarden, Google, Proton etc.  
---
title: Setting up authentication using PAM_U2F and Yubikeys
description: "How I set up up Sudo, Lock Screen and Login authentication via a Yubikey using Pam_u2F"
date: 2026-06-1T20:16:15.698Z
preview: ""
draft: true
tags: [Yubikey, PAM, Bazzite, sudo, login,]
categories: []
showComments: true
---

**Introduction**

My original aim for this project was to enable a Windows Hello-style experience for unlocking or logging into my Bazzite desktop. This would be achieved using either a USB fingerprint reader or a webcam with a built-in IR camera for Windows Hello.
Unfortunately, the USB fingerprint reader I own, based on the Focal Systems FT9201, has limited ability to work with Bazzite — or Linux-based operating systems in general — due to a lack of proper, stable driver support. There are a couple of projects that attempt to provide a working driver (https://github.com/banianitc/ft9201-fingerprint-driver/ and https://github.com/ryenyuku/libfprint-ft9201), but both had numerous show-stopping issues for me personally.
Another blow: Howdy, the facial recognition program for Linux-based operating systems, would need to be installed via a Fedora third-party COPR repository (COPR is a Fedora community-built package hosting service). This would need to be layered onto Bazzite, but because Bazzite is mostly immutable (meaning the system is provided as a read-only image that is updated as a whole), layered third-party packages can occasionally be broken by system updates.
That didn't sound like the maintenance-free, easy solution I was looking for. It is probably possible to get it working, and hopefully someday a more officially supported installation method will become available.

**The Solution — YubiKey**
Thankfully, I had another solution: the YubiKey.
This is a hardware authentication device, a USB-stick-sized gadget that can be registered with a service such as Google or Bitwarden. During registration, the key generates a unique public/private key pair for that service. It sends the public half to the service to keep on file, while the private half remains locked inside the key and never leaves it (nor can it be extracted).
When you next log in to a service with a YubiKey registered — Google, for example — you can use it as your second factor after entering your password. The browser sends a cryptographic challenge to the key over USB, you touch the gold contact to give consent (this can be disabled, but I don't recommend it as it as otherwise malware could easily use the key to authenticate, reducing security), and the key signs the challenge with its private key before sending the signed response back.
Modern standards such as FIDO2 also support optional user verification through a PIN or a fingerprint on devices like the YubiKey Bio. This means that even if your key were stolen, it could not be used without that additional factor. After too many failed attempts, the FIDO application may become blocked and require recovery or reset procedures, so choose your PIN carefully and keep it safe.
For my use case, the main advantage over the USB fingerprint reader or Howdy is that the YubiKey has native support on Bazzite through a tool called pam-u2f, which is available in the standard Fedora repositories. Installation is therefore straightforward. There's no fighting with Bazzite's immutable design, and edits to PAM configuration files such as /etc/pam.d/sudo are persistent, so updates to Bazzite should not break this setup.
YubiKey also provides a Linux command-line management utility called ykman.

**YubiKey Pre-Setup and Things to Bear in Mind**
Even though a YubiKey configured for login or sudo will still allow password fallback (provided you configure PAM appropriately), I strongly recommend purchasing a second key as a backup. I also recommend maintaining a strong password and storing a copy in a secure location such as a safe.
This ensures that you should never be permanently locked out of your machine.
The other thing I would recommend is setting a fixed hostname. FIDO2 credentials are associated with an origin, and in the case of pam-u2f that origin is derived from the hostname. If the hostname changes, authentication can break.
Because Bazzite did not automatically maintain a fixed hostname on my installation, various events could override it and break YubiKey authentication for login, sudo, and other PAM-protected services.
I encountered this issue after dual-booting into Windows. My router assigned my Windows hostname to the IP address, and Bazzite subsequently adopted that hostname when I booted back into Linux. This caused my pam_u2f configuration to stop working until I restored the original hostname.I used 
sudo hostnamectl set-hostname your-name
to set a fixed hostname.
If you use a Chromium-based browser (I tested Edge, Chrome, and Chromium), I do not recommend replacing your primary graphical login password with YubiKey authentication. These browsers use KDE Wallet to store encrypted cookies and credentials. KDE Wallet is normally unlocked using your login password; if you bypass password authentication entirely, KDE Wallet may remain locked.
In my testing, this resulted in browsers entering an infinite page-loading loop unless Incognito Mode was used.
Firefox uses its own encrypted storage mechanism and was unaffected.
One final point: YubiKey firmware cannot be upgraded. This is an intentional security design decision. Don't purchase an older model expecting to update it later. For the setup described in this article, you'll want a YubiKey with firmware version 5.2.3 or newer.


**How I Set Up My YubiKeys**

**Initial Setup**

First, you'll need to install ykman (Yubico's key-management tool):
which ykman || sudo rpm-ostree install yubikey-manager
Plug in your key and confirm it's recognised:
ykman info
Then change the PIN on your key, for the cases where FIDO requires user verification:
ykman fido access change-pin
Next, install pam-u2f:
sudo rpm-ostree install pam-u2f
Reboot, then verify it landed:
ls /usr/lib64/security/pam_u2f.so
You'll also want libfido2 (pre-installed in my case — check with rpm -q libfido2 and install it with sudo rpm-ostree install libfido2 if it isn't there).
pam-u2f (Pluggable Authentication Module — Universal 2nd Factor) is the module that allows authentication via a USB key like the YubiKey; libfido2 is the underlying library that implements the FIDO2 protocol.
After installation, your key (or keys) need to be enrolled with pam-u2f:
mkdir -p ~/.config/Yubico
pamu2fcfg > ~/.config/Yubico/u2f_keys
If you have two keys, run the following for the second key:
pamu2fcfg -n >> ~/.config/Yubico/u2f_keys
This appends the second key to the same line as the first, so pam-u2f treats them as alternatives. Verify with:
cat ~/.config/Yubico/u2f_keys
It should look like your username, a colon, a long string of numbers and letters, another colon, then another long string. The long strings are the key credentials that PAM will use to authenticate your keys.

A quick note, if you wanted user verification, for example a pin code or for a biometric yubikey, a fingerprint, then adding –v to pamu2fcfg > ~/.config/Yubico/u2f_keys, would add this as a requirement. I did not do this as these keys will never leave my house, so the added convenience is fine. When I add keys to my laptop however, I will be using user verification however. 
Step 3: SAFETY
Whenever you're messing around with authentication files, open a second terminal and run sudo -i (the -i ensures that the sudo privileges won't expire until the terminal is closed). This is in case something gets screwed up and you can no longer authenticate as root; the open terminal allows you to revert your changes.
Before editing any files, please check that your TTY interface works, this is an additional fallback. Press ctrl + alt + f3, this load TTY mode, type in your user and password to check if they work, then press ctrl +alt + f2 to go back. This will allow you to revert any changes made if something breaks. 
Allowing the YubiKey to authenticate
I started with sudo, as that would most easily allow me to test if this was working. I ran:
sudo nano /etc/pam.d/sudo
This brings up a text file with rows that look like auth include system-auth and account include system-auth.
As the first line I added:
auth sufficient pam_u2f.so cue
This contains all the instructions needed. The YubiKey is at the top so it will be tried first; auth means the module is being used for authentication; sufficient means if this module suceeds, grant access, if it fails, ignore and move onto the next line, pam_u2f.so is the module doing the authentication; cue tells it to prompt you to touch the key so you know it's waiting.
Ctrl+O to save, Enter to confirm, then Ctrl+X to exit. To test if it worked, run:
sudo -k
sudo whoami
It should ask you to touch the key to authenticate, then reply with root. Repeat with the second key if you have one. Then test again with only your password to confirm that the sufficient line passes through to the next.

In my case it was successful, so I proceeded to /etc/pam.d/kde (for the lock screen) and /etc/pam.d/kscreensaver (mostly for any legacy programs that reference this file). Same method as above:
sudo nano /etc/pam.d/kde
Add auth sufficient pam_u2f.so cue as the top line, Ctrl+O, Enter, Ctrl+X, then test both keys and password by locking your screen.
For polkit, in my case the configuration file lived under /usr/lib/pam.d/polkit-1, which would not persist between updates. Thankfully, copying the file to /etc/pam.d/polkit-1 will make it persistent:
sudo cp /usr/lib/pam.d/polkit-1 /etc/pam.d/polkit-1
sudo nano /etc/pam.d/polkit-1
Add auth sufficient pam_u2f.so cue at the top. Test with:
pkexec ls /root
This should produce a dialogue box, your key should start to flash, then when pressed it should list the root directory contents. 
If you wish to make the Yubikey a requirement for authentication, then you could change sufficient to required. With required, even if a later password check succeeds, the overall authentication still fails — so losing the key really does lock you out, and recovery means dropping to a TTY and editing the modified files to remove the pam_u2f line. I don’t recommend this as it means to if you lose your key, then authenticating back in would require editing the modified files using TTY to remove the pam_u2f line. 

Logging in with the YubiKey

My version of Bazzite was using plasmalogin for login management, which is a newer system than SDDM. plasmalogin has three files used for authentication: plasmalogin, plasmalogin-greeter, and plasmalogin-autologin. I edited plasmalogin. k. To make restoring easier, I copied my plasmalogin file
sudo cp /usr/lib/pam.d/plasmalogin /etc/pam.d/plasmalogin 
The /etc/ version of the file is preferred by Bazzite, and would be persistent, so I edited that file but, if deleted the system would resort back to /usr/lib/pam.d/plasmalogin. Then
sudo nano /etc/pam.d/plasmalogin
to edit the file, however there is a catch with this file, the 1st line is
auth [success=done ignore=ignore default=bad] pam_selinux_permit.so
This is related to SELinux (Security-Enhanced Linux, a kernel security module that handles mandatory access control), which I left alone — too much risk of breaking something. Add auth sufficient pam_u2f.so cue as the second line, immediately after the SELinux one. This is where KDEwallet partially breaks, because sufficient passes as soon as the check is complete, then the line line to authenticate KDE wallet never runs. 
After that, logout, select user and in my case it won’t try and use the Yubikey straight away, instead it shows the password field, this is skippable by just pressing enter, then it activates the yubikey. Test your backup key if you have one, then password. If the login fails, press ctrl + alt + f3, then run 
sudo rm /etc/pam.d/plasmalogin
To delete the edited file.

Conclusion

Other than the the dual-boot hostname incident and the kdewallet issues mentioned above, and the login screen loading password field first, this setup has been flawless. It has allowed me to set a much more secure root password, as I don;t have to type it every single time I want to authenticate. I have also added my Yubikeys as a 2nd factor to a number of online services I use, such as Bitwarden, Google, Proton etc.  
