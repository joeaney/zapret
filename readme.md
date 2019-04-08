# ﻿zapret v.20

As a rule, DPI tricks do not help to bypass https blocking. What is it for?

-----------------
Bypass the blocking of web sites http. 
----------------


## How it works

DPI providers have gaps. They happen from what the DPI rules write for ordinary user programs, omitting all possible cases that are permissible by standards.
This is done for simplicity and speed. It makes no sense to catch hackers, of which 0.01%, because all the same, these locks are quite simple, even by ordinary users.

Some DPIs cannot recognize the http request if it is divided into TCP segments.  
For example, a query like `GET / HTTP/1.1\r\nHost: kinozal.tv......`  
we send in 2 parts: first comes `GET ", затем "/ HTTP/1.1\r\nHost: kinozal.tv.....`.  
Other DPI stumble when heading `Host:" spelled in another case: eg, "host:`.  
In some places, adding additional space after the method works: `GET /" => "GET  /` or adding a dot at the end of the host name: `Host: kinozal.tv.`

## How to put this into practice in the linux system

How to make the system break the request into parts? You can run the entire TCP session through transparent proxy, or you can replace the tcp window size field on the first incoming TCP packet with a SYN, ACK.

Then the client will think that the server has set a small window size for it and the first data segment will send no more than the specified length. In subsequent packages, we will not change anything.

The further behavior of the system at the choice of the size of the sent packets depends on the algorithm implemented in it. Experience shows that linux first package always sends no more than the length specified in the window size, the rest of the packets up to some time sends no more than max (36, specified_size).

After a certain number of packets, the window scaling mechanism is triggered and the scaling factor starts to be taken into account, the packet size becomes no more than max (36, specified_ramer << scale_factor).

Not very elegant behavior, but since we do not affect the size of the incoming packet, and the amount of data received via http is usually much higher than the amount sent, then only small delays will appear visually.

Windows behaves in a similar case much more predictably. The first segment goes the specified length, then the window size changes depending on the value sent in the new tcp packets. That is, the speed is almost immediately restored to a possible maximum.

To intercept a packet from SYN, ACK does not represent any complexity by means of iptables.  

However, the options for editing packages in iptables are severely limited.  

It’s just not possible to change window size with standard modules.  

For this, we will use the NFQUEUE tool. This tool allows packets to be processed by processes running in user mode.  

The process, accepting a package, can change it, which is what we need.

`iptables -t raw -I PREROUTING -p tcp --sport 80 --tcp-flags SYN,ACK SYN,ACK -j NFQUEUE --queue-num 200 --queue-bypass`

It will give the packages we need to the process that listens on the queue with the number 200.

It will replace the window size. PREROUTING will catch both packets addressed to the host itself and routed packets. That is, the solution works the same way on the client and on the router. On a PC-based or OpenWRT router.

In principle, this is enough.  
However, with such an impact on TCP there will be a slight delay.  
In order not to touch the hosts that are not blocked by the provider, you can make such a move.  
Create a list of blocked domains or download it from rublacklist.  
Secure all domains in ipv4 addresses. To drive them into the ipset with the name "zapret".  

Add to rule:

`iptables -t raw -I PREROUTING -p tcp --sport 80 --tcp-flags SYN,ACK SYN,ACK -m set --match-set zapret src -j NFQUEUE --queue-num 200 --queue-bypass`

Thus, the impact will be made only on ip addresses related to blocked sites.  
The list can be updated via cron every few days.  
If you update via rublacklist, then it will take quite a long time. More than an hour. But this process does not take away resources, so it will not cause any problems, especially if the system is constantly running.

If DPI doesn’t get by with splitting a request into segments, then sometimes a change occurs.
"Host:" on "host:". In this case, we may not need a window size replacement, so the chain PREROUTING we do not need. Instead, we hang on outgoing packets in the POSTROUTING chain:

`iptables -t mangle -I POSTROUTING -p tcp --dport 80 -m set --match-set zapret dst -j NFQUEUE --queue-num 200 --queue-bypass`

In this case, additional points are also possible. DPI can catch only the first http request, ignoring subsequent requests in the keep-alive session. Then we can reduce the load on the percent, abandoning the processing of unnecessary packages.

`iptables -t mangle -I POSTROUTING -p tcp --dport 80 -m connbytes --connbytes-dir=original --connbytes-mode=packets --connbytes 1:5 -m set --match-set zapret dst -j NFQUEUE --queue-num 200 --queue-bypass`

It happens that the provider monitors the entire HTTP session with keep-alive requests. In this case, it is not enough to restrict the TCP window when establishing a connection. You must send each new request with separate TCP segments. This task is solved through the full proxying of traffic through a transparent proxy (TPROXY or DNAT). TPROXY does not work with connections originating from the local system, so this solution is applicable only on the router. DNAT works with local connections, but there is a danger of entering into infinite recursion, so the daemon is started as a separate user, and DNAT is disabled for this user via `-m owner`. Full proxying requires more processor resources than manipulating outgoing packets without reconstructing a TCP connection.

```
iptables -t nat -I PREROUTING -p tcp --dport 80 -j DNAT --to 127.0.0.1:1188
iptables -t nat -I OUTPUT -p tcp --dport 80 -m owner ! --uid-owner tpws -j DNAT --to 127.0.0.1:1188
```

## nfqws

This program is a packet modifier and a NFQUEUE queue handler.
It takes the following parameters:

- `--daemon`: demonize prog
- `--qnum=200`: queue number
- `--wsize=4`: change tcp window size to specified size
- `--hostcase`: change the case of the "Host:" header to "host:" by default.
- `--hostnospace`: remove the space after the "Host:" and move it to the end of the value "User-Agent:" to save the packet length
- `--hostspell=HoST`: the exact spelling of the Host header (you can use "HOST" or "HoSt"). automatically includes `--hostcase` The manipulation parameters can be combined in any combination.

## tpws

tpws - This is a transparent proxy.

- `--bind-addr`: what address to listen to. maybe ipv4 or ipv6 address. if not specified, then listens on all ipv4 and ipv6 addresses
- `--port=<port>`: which port to listen to
- `--daemon`: demonize prog
- `--user=<username>`: change the uid of the process
- `--split-http-req=method|host`: way to split http requests into segments: around the method (GET, POST) or around the Host header
- `--split-pos=<offset>`: divide all messages into segments in the specified position. If sending is longer than 8Kb (receive buffer size), each block will be divided by 8Kb.
- `--hostcase`: change the case of the "Host:" header. default to "host:".
- `--hostspell=HoST`: the exact spelling of the Host header (you can use "HOST" or "HoSt"). automatically includes `--hostcase`
- `--hostdot`: Add a dot after the host name: `Host: kinozal.tv.`
- `--hosttab`: add tab after hostname: `Host: kinozal.tv \ t`
- `--hostnospace`: remove space after "Host:"
- `--methodspace`: add space after method: "GET /" => "GET  /"
- `--methodeol`: add line feed before method: "GET /" => "\r\nGET  /"
- `--unixeol`: convert 0D0A to 0A and use everywhere 0A
- `--hostlist=<filename>`: act only on domains included in the list of filename. subdomains are automatically counted. The file must have a host on each line. The list is read once at the start and is stored in the memory in the form of a hierarchical structure for a quick search. for a list of RKN, a system with 128 Mb of memory may be required! calculate the RAM requirement for the process as 3-5 times the size of the list file. on the HUP signal, the list will be re-read with the next accepted connection. The manipulation parameters can be combined in any combination.

There are exceptions: `split-pos` replaces `split-http-req`. hostdot and hosttab are mutually exclusive.

## Providers

Since the situation with blocking on individual providers may change, information may become outdated. It is given more for example than direct guidance.

The author does not monitor and promptly update this information.

- **mns.ru**: need to replace window size by 3. mns.ru removes blocked domains from issuing their DNS servers. we change for third-party. uplink westcall banned by IP address from the list of RKN where https is present
- **at-home.ru**: With the default connection, everything was blocked by IP. after ordering an external IP (static NAT), they are banned by IP https addresses. To bypass the DPI, the replacement of windows size by 3 works, but instability and suspensions have been noticed. Split request works best around the method during the entire http session. In https certificate is replaced. If you have everything blocked by IP, then there is no way other than proxying port 80 by analogy with 443.
- **beeline (corbina)**: need to replace the register "Host:" throughout the http session. For some time, "host" does not work, but other letter registers work.
- **dom.ru**: HTTP sessions should be proxied through tpws with the replacement of the "Host:" register and the separation of TCP segments on the "Host:" header. Achtung! Domra blocks all subdomains of a blocked domain. It is impossible to find out the IP addresses of various subdomains from the registry of locks, so if suddenly a blocking banner comes out of a website, go to the firefox console, the network tab. Download the site and see where the redirect goes. Then put the domain in zapret-hosts-user.txt. For example, on kinozal.tv there are 2 requested subdomains: s.kinozal.tv and st.kinozal.tv with different IP addresses. Domra intercepts DNS queries and pops up his false answer. This is managed through a false-answer drop via iptables by the presence of the IP address of the stub or via dnscrypt.
- **sknt.ru**: checked with tpws with the parameter `--split-http-req = method`. the provider monitors each packet, so when using nfq, the 2nd request in the same session hangs
- **Rostelecom/tkt**: http division request helps for segments, mns.ru settings are suitable TKT was purchased by Rostelecom, Rostelecom filtering is used. Since DPI does not discard the incoming session, but only pops up its packet, which comes before the response from the real server, locks also do without the use of "heavy artillery" by the following rule: `iptables -t raw -I PREROUTING -p tcp --sport 80 -m string --hex-string "|0D0A|Location: http://warning.rt.ru" --algo bm -j DROP --from 40 --to 200`
- **tiera**: Requires split http requests throughout the session.

## Other providers

The first step is to find out if your DNS provider will replace it.  
Look at what the blocked hosts have with your provider and through some web net tools that you can google a lot. Compare.  
If the answers are different, then try to register the same hosts from the DNS server 8.8.8.8 through your provider.  
If the answer from 8.8.8.8 is normal - change the DNS. If the answer is not normal, then the provider intercepts requests to third-party DNS.  
Use dnscrypt.

Next, you need to find out which DPI bypass method works on your provider.  
The `https://github.com/ValdikSS/blockcheck` script will help you with this.  
Choose which daemon to use: nfqws or tpws.  
Manually prepare iptables rules for your case, run them.  
Start the daemon with the necessary parameters manually.  
Check if it works.  
When you find the working version, edit the init script for your system.  
Uncomment ISP = custom. Add your code to the place "# PLACEHOLDER" by analogy with the sections for other providers for the found working combination.  
For openwrt, place your code in /etc/firewall.user by analogy with ready-made scripts.

## Ways to get a list of blocked IP

1. Enter the blocked domains in `ipset/zapret-hosts-user.txt` and run `ipset/get_user.sh`
At the output, get `ipset/zapret-ip-user.txt` with IP addresses.

2. `ipset/get_reestr.sh` receives a list of domains from rublacklist and then resolves them to ip addresses in the `ipset/zapret-ip.txt` file. There are ready-made IP addresses in this list, but judging by everything, they are there exactly in the form that RosKomPozor enters the register. Addresses can change, shame does not have time to update them, and providers rarely banned by IP: instead, they ban http requests with the “bad” Host: heading regardless of the IP address. Therefore, the script resolves everything itself, although it takes a lot of time.  
An additional requirement is the amount of memory in `/tmp` to save the downloaded file there, which is several MB in size and continues to grow. On `openwrt/tmp` routers, it is tmpfs, that is, ramdisk.  
In the case of a router with 32 MB of memory, it is not enough, and there will be problems. In this case, use the following script.

3. `ipset/get_anizapret.sh`. quickly and without load on the router receives a sheet with `antizapret.prostovpn.org`.

4. `ipset/get_combined.sh`. for providers that block IP by https, and the rest by DPI. IP https is entered in ipset ipban, the rest in ipset zapret.
Since the large list of RKNs is downloaded, the space requirements in `/tmp` are similar **2.**

All variants of the considered scripts automatically create and fill in ipset.  
Options 2-4 additionally call option 1.

On routers, it is not recommended to call these scripts more often once in 2 days, since saving goes either to the internal flash memory of the router or, in the case of extroot, to a flash drive.

In both cases, too frequent recording can kill the flash drive, but if this happens with the internal flash memory, then you just kill the router.

Forced ipset update executes the `ipset/create_ipset.sh` script.
The ILV list has already reached an impressive size of hundreds of thousands of IP addresses. Therefore, the ip2net utility is used to optimize ipset. It takes a list of individual IP addresses and tries to find in it subnets of the maximum size (from / 22 to / 30) in which more than 3/4 of the addresses are blocked. ip2net is written in C because the operation is resource intensive. Other ways the router can not pull.
If ip2net is compiled or a binar is copied to the ip2net directory, the create_ipset.sh script uses an ipset of the hash: net type, running the list through ip2net.
Otherwise, ipset of hash: ip type is used, the list is loaded as is.
Accordingly, if you don’t like ip2net, just remove the binarks from the ip2net directory.

You can add a list of domains to `ipset/zapret-hosts-user-ipban.txt`. Their ip addresses will be placed in a separate ipset "ipban". It can be used to force all connections to a transparent proxy "redsocks" or to a VPN.

## Domain Name Filtering

An alternative to ipset is to use tpws with a list of domains.
The list of domains of the RKN can be obtained by the script `ipset/get_hostlist.sh` - put in `ipset/zapret-hosts.txt`.
This script automatically adds domains from `zapret-hosts-user.txt` to the RTL list.
tpws should run without filtering by ipset. All http traffic goes through tpws, and it decides whether to use stupor depending on the Host: field in the http request.
This creates an increased load on the system.
The domain search itself works very quickly, the load is connected with pumping the amount of data through the process.
The option is well suited for those who have a fast system with 128+ MB of memory and the provider uses DPI.

## Installation example for debian 7

Debian 7 initially contains 3.2 kernel. It does not know how to do DNAT on localhost.
Of course, you can not bind tpws to 127.0.0.1 and replace in the rules of iptables "DNAT 127.0.0.1" with "REDIRECT", but it is better to install a fresher core. It is in a stable repository:

```bash
apt-get update
apt-get install linux-image-3.16
```

Install packages:

```bash
apt-get update
apt-get install libnetfilter-queue-dev ipset curl
```

Copy the "zapret" directory to `/opt`.

Collect nfqws:

```
cd /opt/zapret/nfq
make
```

Assemble tpws:

```
cd /opt/zapret/tpws
make
```

Build ip2net:
```
cd /opt/zapret/ip2net
make
```

Copy `/opt/zapret/init.d/debian7/zapret` to `/etc/init.d`.

In `/etc/init.d/zapret` select "ISP". Depending on it, the necessary rules will be applied.

In the same place, select the SLAVE_ETH parameter corresponding to the name of the internal network interface.

Enable autostart: chkconfig zapret on (optional) Manually get a new list of ip addresses for the first time: `/opt/zapret/ipset/get_antizapret.sh`
Stopping a sheet update task:

```
crontab -e
```

Create the line `0 12 * * * / 2 /opt/zapret/ipset/get_antizapret.sh`. This means at 12:00 pm every 2 days to update the list.

Start service: `service zapret start`

Try to go somewhere: http://ej.ru, http://kinozal.tv, http://grani.ru.

If it does not work, then stop the zapret service, add the rule to iptables manually, run nfqws in the terminal under the root with the necessary parameters.


Trying to connect to blocked sites, watch the output of the program.

If there is no response, then the wrong queue number is most likely indicated, or the destination ip is not in the ipset.

If there is a reaction, but the blocking is not bypassed, then the bypass parameters selected are incorrect, or this tool does not work in your case on your provider.
Nobody said it would work everywhere.

Try taking a dump in wireshark or `tcpdump -vvv -X host <ip>`, see if the first TCP segment really goes short and whether the "Host:" register changes.

## ubuntu 12, 14

There is a ready config for upstart: `zapret.conf`. It needs to be copied to `/etc/init` and configured by analogy with debian.
Start service: `start zapret`
Stop service: `stop zapret`
Viewing messages: `cat /var/log/upstart/zapret.log`
Ubuntu 12, like debian 7, has a 3.2 kernel. See the note in the "debian 7" section.

## ubuntu 16,debian 8

The process is similar to debian 7, but you need to register the init scripts in systemd after copying them into /etc/init.d.
By default, lsb-core may not be installed.

```
apt-get update
apt-get --no-install-recommends install lsb-core
```

- install: `/usr/lib/lsb/install_initd zapret`
- remove: `/usr/lib/lsb/remove_initd zapret`
- start: `sytemctl start zapret`
- stop: `systemctl stop zapret`
- status, output messages: `systemctl status zapret`

## Other linux systems

There are several main service startup systems: sysvinit, upstart, systemd.

The setting depends on the system used in your distribution.

A typical strategy is to find a script or configuration to launch other services and write your own by analogy, read the launch system documentation if necessary.

The necessary commands can be taken from the proposed scripts.

## Faervalli

If you use some kind of firewall management system, then it may conflict with an existing startup script. In this case, the rules for iptables should be bolted to your firewall separately from the tpws or nfqws startup script.

This is how the issue is solved in the case of openwrt, since there is a firewall control system there.

When re-applying the rules, it could break the iptables settings made by the script from init.d.

## What to do with `openwrt/LEDE`

Install additional packages:

```
opkg update
opkg install iptables-mod-extra iptables-mod-nfqueue iptables-mod-filter iptables-mod-ipopt ipset curl bind-tools
opkg install kmod-ipt-raw # for new LEDE
```

The main difficulty is to compile programs on C. This can be done on linux x64 using the SDK, which can be downloaded from the official openwrt website or LEDE. But the process of cross-compiling is always a challenge.

It is not enough to run make as on a traditional linux system.

Therefore, binaries have ready-made static binaries for all the most common architectures.

Static build means that the binarik does not depend on the type of libc (glibc, uclibc or musl) and the presence of installed so - it can be used immediately.
Just to fit the type of CPU. ARM and MIPS have several versions. Find the option that works on your system.

Most likely there is such. If not, you will have to collect it yourself.

copy the directory `zapret` in `/opt` to the router.

Copy the working nfqws binaries to `/opt/zapret/nfq`, tpws to `/opt/zapret/tpws`, ip2net to `/opt/zapret/ip2net`.

Copy `/opt/zapret/init.d/zapret` to `/etc/init.d`.

In `/etc/init.d/zapret` select "ISP". Depending on it, the necessary rules will be applied.

```
/etc/init.d/zapret enable
/etc/init.d/zapret start
```

Depending on your provider, make the necessary entries in `/etc/firewall.user`.

`/etc/init.d/firewall restart`

View through `iptables -L` or through `luci` tab "firewall" whether the necessary rules appeared.

Stopping a sheet update task:

`crontab -e`

Create the line `0 12 * * * / 2 /opt/zapret/ipset/get_antizapret.sh`. This means at 12:00 pm every 2 days to update the list.

## Https blocking bypass

As a rule, DPI tricks do not help to bypass https blocking.

You have to redirect traffic through a third-party host.

It is proposed to use a transparent redirect through socks5 through iptables + redsocks, or iptables + iproute + openvpn.

Setting the option from redsocks to openwrt is described in https.txt.