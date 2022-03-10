# Purpose
There are many guides and cheatsheets out there that mention the commands needed to implement pivoting, althought, they tend to be confusing and many of them lack a few important notes about what is really happening during the process. This is an attemp to make a clear guide focused on the use of pivoting in penetration testings / CTF challenges.

# Important Notes

- **Do not use ICMP echo requests (ping) to test SOCKS proxies**.  
SOCKS is an Internet protocol that exchanges network packets between a client and server through a proxy server. Practically, a SOCKS server proxies TCP connections to an arbitrary IP address, and provides a means for UDP packets to be forwarded. 
You should test it using a TCP based protocol (e.g. try to SSH or send an HTTP GET request at a host accross the pivot tunnel).

- **NMAP must be used with TCP connect scan (-sT) and no ping (-Pn)**  
This also applies to version (-sV) and script scans (-sC). They should be used in combination with -sT and -Pn.  
**Examples:**  
```
proxychains nmap -sT -Pn -p- x.x.x.x
proxychains nmap -sT -Pn -sV -sC -p 21,80,443,445 x.x.x.x
```

- **Using scripts and binaries with proxychains**  
One tip for using proxychains is to ensure that if you are running an interpreted program (like a Python script) its a good idea to explicitly reference the Python binary before that script, even if the script starts with a hash bang, e.g.:  
```
proxychains4 [-q -f proxychains.conf] python python_script.py
```

  Without this specific reference to the script interpreter, sometimes the traffic generated from the script will fail to be routed through the proxy as you intended, and the network connection will fail. [Tip Source](https://thegreycorner.com/2021/12/15/hackthebox_dante-review.html)

- **Uncomment line "quite mode" in /etc/proxychains.conf to avoid stdout that can be frustrating at times.**  
This is merely a suggestion.


# Pivoting Guide
## SSH Dynamic Port Forwarding
Allows you to create a socket on the local (ssh client) machine, which acts as a SOCKS proxy server. When a client connects to this port, the connection is forwarded to the remote (ssh server) machine, which is then forwarded to a dynamic port on the destination machine.

**How to set it up**:  
1. Edit /etc/proxychains.conf and implement the following:
   - Remove **Dynamic chain** from comment.
   - Comment **Strict chain** and **Random chain**.
   - Append line **socks4 127.0.0.1 9050** at the end of the document (proxy list), save and close file. *You can, of course, use a different port.

2. Setup the SSH Dynamic Port Forwarding:  
`ssh -D 127.0.0.1:9050 user@victim-IP`

**Usage examples**:  
With x.x.x.x being the ip address of a host that belongs to the tunneled network:  
`proxychains nmap -sT -Pn -p- x.x.x.x`  

`proxychains smbmap -H x.x.x.x`  

`proxychains ssh user@x.x.x.x`  

In order to use firefox throught the tunnel:  

`proxychains firefox`  

## SSH Remote Port Forwarding  
If you are searching how to get a **reverse shell through a pivot tunnel**, this is what you need. It allows you to forward a port on the remote (victim) machine to a port on the local (attacker) machine.

**How to set it up**:  
1. SSH into the victim machine.
2. Edit /etc/ssh/sshd_config and implement the following:
   - Remove **GatewayPorts no** from comment and change it to **Yes**. Save and close file. 
 
   **This is very important and many guides around the web do not mention it. If you don't do this you will only be able to set the tunnel on 127.0.0.1 rather than      0.0.0.0, which will end up not forwarding traffic when it originates from any host except localhost.*

   - Restart the SSH service for changes to be effective. `sudo service ssh restart`
   - Exit and return to your machine.

3. Setup the SSH Remote Port Forwarding:  
Essentially, after setting it up as mentioned, the command is as simple as that:  
`ssh -R 2222:*:2222 user@victim-IP`

In order to test if it works, you could do the following:  
Set a listener (e.g. netcat) on your attacker machine (on the port you configured the remote forward, in this example 2222) and make a request from the victim machine to itself (localhost). In this example, that would be `nc 127.0.0.1 2222`. If your attacker machine receives the connection then it means that a) it works and b) every connection from pivoting hosts to the victimip:2222 will be forwarded to attacker machine.  

You can set multiple ports like this:  
`ssh -R 2222:*:2222 -R 3333:*:3333 user@victim-IP`  

**Caution**: Requests from foreign hosts (pivoting network) must be addressed to the victim ip in order to be forwarded back to the attacker machine.

**You can also implement Remote Port Forwarding by SSH connecting from victim to attacker machine.*

## SSH Local Port Forwarding  
Local port forwarding allows you to forward a port on the local (attacker) machine to a port on the remote (victim) machine. Particularly usefull in order to scan local ports on victim.  
  
**Usage**:  
`ssh user@victim-IP -L 8888:127.0.0.1:8086`  
  
You can now use e.g. nmap to scan port 8086 on the victim machine like this:  
`nmap -Pn -n -p8888 -sV 127.0.0.1 `  

## Double Pivoting
An example of how to achieve double pivoting with SSH Dynamic Port Forwarding and Proxychains.  
  
**Concept**:  
Suppose we have the following 4 machines.  
| IP          |   Role    |  
|-------------|:---------:|
| 10.10.10.10 | Attacker  |
| 10.10.10.11 | Jumphost1 |
| 172.16.1.12 | Jumphost2 |
| 172.16.2.13 | Jumphost3 |
  
Attacker can reach Jumphost1.  
Jumphost1 can reach Jumphost2.  
Jumphost2 can reach Jumphost3.  

1. In order to reach Jumphost2, implement **SSH Dynamic Port Forwarding** instructions mentioned before, against Jumphost1.
2. In order to reach Jumphost3, edit /etc/proxychains.conf and add another proxy entry at the end of the document. The end of your proxychains.conf will now look something like this:  
```
...
socks4 127.0.0.1 9050
socks4 127.0.0.1 9999
```  
3. SSH into Jumphost1 and set another Dynamic Port Forwarding against Jumphost2 (the local port you choose must be the same as the one set for your second proxy entry in proxychains.conf):  
`ssh -D 127.0.0.1:9999 user@Jumphost2`  

You should now be able to reach Jumphost3.

## sshuttle - A transparent proxy-based VPN using SSH  
sshuttle allows you to create a VPN connection from your machine to any remote server via ssh, as long as that server has python 2.3 or higher. To work, you must have root access on the local machine, but you can have a normal account on the server. It's  valid  to  run  sshuttle  more  than once simultaneously on a single client machine, connecting to a different server every time, so you can be on more than one VPN at once. Check the [sshuttle github repository](https://github.com/sshuttle/sshuttle).

**Usage**:  
Assuming we want to pivot into 172.16.2.0/16:  
`sshuttle -vvr root@victim 172.16.2.0/16`

If you want to use ssh key:  
`sshuttle -vvr root@victim --ssh-cmd 'ssh -i ~/.ssh/id_rsa' 172.16.2.0/16`


## chisel   
Chisel is a fast TCP/UDP tunnel, transported over HTTP, secured via SSH. Single executable including both client and server. Written in Go (golang). It's insanely awesome and useful.

**Instalation**:  
You can easily install chisel on kali:  
`apt install chisel`  

In order to use it, you also need to upload the chisel binary on the victim. You can download precompiled versions [here](https://github.com/jpillora/chisel/releases).  
  
**Local port forwarding example**  
On your attacker machine:  
`chisel server -p 8000 --reverse`  
  
On the victim machine:  
`./chisel_1.7.7_linux_amd64 client attacker-ip:8000 R:1234:127.0.0.1:8443`  
  
This is going to forward traffic from attacker machine port 1234 to victim machine port 8443.


## Using Burpsuite as Proxy  
Burpsuite supports the capability of setting proxies, an incredibly useful and powerfull feature.

**How to set it up**:  
Fire up Burpsuite and implement the following:
- Goto Proxy/Options and hit **add** on the Proxy Listeners section.
- Set a binding port and interface.
- on the second tab, set the host and port you want to redirect traffic to.

In combination with setting up an SSH Dynamic Port Forwarding or sshuttle, you can now use Burpsuite to pivot traffic to desired hosts by sending traffic to your localhost bind port. A useful example with gobuster dir brute through the tunnel (Assumming that you set port 2222 as the redirect port):  
`gobuster dir -u http://127.0.0.1:2222 -t 40 -w /some/dirlist.txt`



## Usefull Guides & References
- https://artkond.com/2017/03/23/pivoting-guide/
- https://catharsis.net.au/blog/network-pivoting-and-tunneling-guide/
- https://medium.com/cyberxerx/how-to-setup-proxychains-in-kali-linux-by-terminal-618e2039b663
