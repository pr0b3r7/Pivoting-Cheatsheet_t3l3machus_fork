## Purpose
There are many guides and cheatsheets out there that mention the commands needed to implement pivoting, althought, they tend to be confusing and many of them lack a few important notes about what is really happening during the process. This is an attemp to make a clear guide focused on the use of pivoting in penetration testings / CTF challenges.

## Important Notes

- **Do not use ICMP echo requests (ping) to test SOCKS proxies**.  
SOCKS is an Internet protocol that exchanges network packets between a client and server through a proxy server. Practically, a SOCKS server proxies TCP connections to an arbitrary IP address, and provides a means for UDP packets to be forwarded. 
You should test it using a TCP based protocol (e.g. try to SSH or send an HTTP GET request at a host accross the pivot tunnel).

- **NMAP must be used with TCP connect scan (-sT) and no ping (-Pn)**  
This also applies to version (-sV) and script scans (-sC). They should be used in combination with -sT and -Pn.  
**Examples:**  
`proxychains nmap -sT -Pn -p- x.x.x.x`  
`proxychains nmap -sT -Pn -sV -sC -p 21,80,443,445 x.x.x.x`

- **Uncomment line "quite mode" in /etc/proxychains.conf to avoid stdout that can be frustrating at times.**  
This is merely a suggestion.


## Pivoting Guide
### SSH Dynamic Port Forwarding
Allows you to create a socket on the local (ssh client) machine, which acts as a SOCKS proxy server. When a client connects to this port, the connection is forwarded to the remote (ssh server) machine, which is then forwarded to a dynamic port on the destination machine.

**How to set it up**:  
1. Edit /etc/proxychains.conf and implement the following:
   - Remove **Dynamic chain** from comment.
   - Comment **Strict chain** and **Random chain**.
   - Append line **socks4 127.0.0.1 9050** at the end of the document (proxy list), save and close file. *You can, of course, use a different port.

2. Setup the SSH Dynamic Port Forwarding:
`ssh -D 127.0.0.1:1234 user@victim-IP`

**Usage examples**:  
With x.x.x.x being the ip address of a host that belongs to the tunneled network:  
`proxychains nmap -sT -Pn -p- x.x.x.x`  

`proxychains smbmap -H x.x.x.x`  

`proxychains ssh user@x.x.x.x`  

In order to use firefox throught the tunnel:  

`proxychains firefox`  

### SSH Remote Port Forwarding  
If you are searching how to get a **reverse shell through a pivot tunnel**, this is what you need. It allows you to forward a port on the remote (ssh server) machine to a port on the local (ssh client) machine, which is then forwarded to a port on the destination machine.

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
**Caution**: Requests from foreign hosts (pivoting network) must be addressed to the victim ip in order to be forwarded back to the attacker machine.

**You can also implement Remote Port Forwarding by SSH connecting from victim to attacker machine.*

### sshuttle - A transparent proxy-based VPN using ssh  
sshuttle allows you to create a VPN connection from your machine to any remote server that. You can connect to via ssh, as long as that server has python 2.3 or higher. To work, you must have root access on the local machine, but you can have a normal account on the server. It's  valid  to  run  sshuttle  more  than once simultaneously on a single client machine, connecting to a different server every time, so you can be on more than one VPN at once. Check the [sshuttle github repository](https://github.com/sshuttle/sshuttle).

**Usage**:  
Assuming we want to pivot into 172.16.2.0/16:  
`sshuttle -vvr root@victim 172.16.2.0/16`

If you want to use ssh key:  
`sshuttle -vvr root@victim --ssh-cmd 'ssh -i ~/.ssh/id_rsa' 172.16.2.0/16`

### Using Burpsuite as Proxy  
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
