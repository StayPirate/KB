public:: true
tags:: sudo, setuid

- ### Check what command the user can run via `sudo`
  *The user password might to be know in order to execute the following commands*
	- ```bash
	  sudo -l
	  ```
	- For semi interactive shells use
	  ```bash
	  sudo -lS
	  ```
- ### Look for unusual setuid executables
	- ```bash
	  find / -perm -4000 2>/dev/null
	  ```