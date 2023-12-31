public:: true

- ```
    .#####.   mimikatz 2.0 alpha (x86) release "Kiwi en C" (Apr  6 2014 22:02:03)
   .## ^ ##.
   ## / \ ##  /* * *
   ## \ / ##   Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
   '## v ##'   https://blog.gentilkiwi.com/mimikatz             (oe.eo)
    '#####'                                    with  13 modules * * */
  ```
- #+BEGIN_NOTE
  It's well known to extract plain-text passwords, hash, PIN code and kerberos tickets from memory. It can also perform pass-the-hash, pass-the-ticket or build Golden tickets.
  #+END_NOTE
- Official [repository](https://github.com/gentilkiwi/mimikatz).
- Official [wiki](https://github.com/gentilkiwi/mimikatz/wiki), additional [docs](https://adsecurity.org/?page_id=1821).
- Download binaries 💾 from official [releases](https://github.com/gentilkiwi/mimikatz/releases), also maintained in [Kali](https://www.kali.org/tools/mimikatz/) at `/usr/share/windows-resources/mimikatz/`.
- Evade detection
	- #+BEGIN_IMPORTANT
	  Due to the mainstream popularity of Mimikatz and well-known detection signatures, consider avoiding using it as a standalone application and use antivirus evasion techniques instead.
	  #+END_IMPORTANT
		- Execute Mimikatz directly from memory using an [injector](https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1) like PowerShell.
		- Use other tools [to dump](https://www.whiteoaksecurity.com/blog/attacks-defenses-dumping-lsass-no-mimikatz/) the entire LSASS process memory and process them with Mimikatz offline.
			- Built-in Task Manager (GUI)
			  logseq.order-list-type:: number
			- [Procdump](https://learn.microsoft.com/en-us/sysinternals/downloads/procdump)
			  logseq.order-list-type:: number
			- [[CrackMapExec]] (`--lsa`)
			  logseq.order-list-type:: number
			- [Lsassy](https://github.com/Hackndo/lsassy)
			  logseq.order-list-type:: number
			- Then process the dump in a secure machine
				- From a trusted Windows machine with Mimikatz
				  ```
				  sekurlsa::minidump lsass.DMP
				  log lsass.txt
				  sekurlsa::logonPasswords
				  ```
				- From a trusted Linux machine with [Pypykatz](https://github.com/skelsec/pypykatz)
				  
				  ```bash
				  pypykatz lsa minidump lsass.DMP
				  ```
- To extract sensible data from the `lsass` process' memory the [`sekurlsa`](https://github.com/gentilkiwi/mimikatz/wiki/module-~-sekurlsa) module can be used. In order to properly work it requires one of the following accounts/permissions.
	- Administrator user. In this case you need to get `debug` privilege via `privilege::debug`.
	  logseq.order-list-type:: number
	- SYSTEM account. In this case `debug` privilege is not needed.
	  logseq.order-list-type:: number
- Plain-text password in Mimikatz output
	- Sometimes Mimikatz returns plain-text passwords instead of their hashes, this is because [[WDigest]] is enabled.
	- That's the case of older operating systems like Windows 7, or operating systems where Wdigest has been manually enabled.
- Abusing TGT and service tickets
	- To dump tickets from LSASS process memory we can use the [`sekurlsa::tickets`](https://github.com/gentilkiwi/mimikatz/wiki/module-~-sekurlsa#tickets) command.
	  {{embed ((65686a95-2f8d-4c16-ba29-4ec6ce616cb4))}}
	- Forge [[Silver Tickets]] 
	  {{embed ((6568a340-887a-4e0e-9515-0344e4728aaa))}}
- Certificates
  id:: 655b7624-0ef9-481b-96a1-2aa762c3f0dd
	- Certificates' private keys are also stored in the LSASS process' memory. As [mentioned here](((655b76db-6fdf-470e-9bea-1d662dcfd6c1))), certificates may be marked as having a non-exportable private key.
	- The [`crypto`](https://github.com/gentilkiwi/mimikatz/wiki/module-~-crypto) module provides capability to either patch the [CryptoAPI](https://learn.microsoft.com/en-us/windows/win32/seccrypto/cryptoapi-system-architecture) function with [`crypto::capi`](https://github.com/gentilkiwi/mimikatz/wiki/module-~-crypto#capi) or the [KeyIso service](http://revertservice.com/10/keyiso/) with [`crypto::cng`](https://github.com/gentilkiwi/mimikatz/wiki/module-~-crypto#cng), **making non-exportable keys exportable**.
- Defeating Mimikatz
	- Windows Defender Credential Guard
		- [Windows Defender Credential Guard](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard) is a security feature that was introduced by Microsoft in **Windows  10 Enterprise** and **Windows Server 2016**. Credential Guard utilizes virtualization-based security and isolated memory management to ensure that only privileged system software can access domain credentials.
		- Once enabled Windows servers and devices in your domain can **no longer use legacy authentication** protocols such as NTLMv1, Digest, CredSSP and MS-CHAPv2, nor can they use Kerberos unconstrained delegation and DES Encryption.
		- According to Mimikatz's author, even though isolated LSAs can’t be queried, Mimikatz can still capture the credentials being entered. If you have control of an endpoint and a privileged user logs in, then it is possible to get the credentials and elevate privileges.
			- {{twitter https://twitter.com/gentilkiwi/status/1044715664823308289}}
		- Starting in Windows 11 version 22H2, VBS *(virtualization-based security)* and Credential Guard are enabled by default on all devices that meet the system requirements. Credential Guard is supported on 64-bit Secure Boot devices only.
	- LSA Protection
		- Starting with *Windows 8.1* and *Windows Server 2012 R2* or later, added protection for the LSA is provided to **prevent reading memory and code injection** by nonprotected processes.
		- Since Windows 11 it's enabled by default.
	- Starting with Windows 8.x and 10, by default, there is no password in memory. Exceptions are:
		- When the DC is unreachable, the kerberos provider keeps passwords for future negotiation.
		- When `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest`, `UseLogonCredential` *(DWORD)* is set to `1`, the wdigest provider keeps passwords.
		- When values in `Allow*` in `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa\Credssp\PolicyDefaults` or `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\CredentialsDelegation`, the CredSSP provider (`tspkgs`) keeps passwords.