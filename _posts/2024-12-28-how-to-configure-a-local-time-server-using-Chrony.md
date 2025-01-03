---
layout: post
title: "How to configure a local time server using Chrony"
categories: linux time-server chrony ntp
---

# How to configure a local time server using Chrony 

One day, an employee was unable to SSH into the jump server of our organization. Upon troubleshooting, I discovered that there is a drastic drift in the time of the jump server. It is at that time, I realized that time synchronization is the heart of servers. When a server's time goes out of sync, authentications won't work, log timestamps cannot be corelated, cron jobs fail and so many other disasters occur. 

So we planned to setup a local time server and configure all the machines of our LAN as NTP clients of the configured local time server. Though it seems to be a straight forward process, there are some tricks that are required to make the NTP clients sync with a local time server, which I'm going to discuss in this blog. 

In my case both the client and server are Ubuntu machines. Chrony is available in the Ubuntu package repository which can be installed using `apt`. The configuration file of Chrony is located in `/etc/chrony/chrony.conf` Let's now jump to the client configuration. 

**Note: I have used the symbols `<>` as placeholders. Make sure that you replace them with your data**

## Configurations

### Chrony client configuration 

We are going to add the following directives to the default configuration file of Chrony. 

```bash
           server <IP_address_of_local_time_server> iburst
           allow <subnet_in_CIDR_notation>
           log measurements statistics tracking
```

The `server` directive specifies the address of the time server which is going to be used as the source of time. We can configure multiple servers as time source for resiliency, however, for simplicity, we are now having a single time source.  `iburst` option is used to expedite time sync initially. 

The `allow` directive allows hosts present in the specified subnet to query time. Though this is a client, this directive becomes when the local time server reboots. 

To monitor and get statistics of time synchronization, we have enabled three different types of logs using the `log` directive. For more information, read the [chrony.conf man page](https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://chrony-project.org/doc/3.4/chrony.conf.html&ved=2ahUKEwjGsabor7GKAxUNxzgGHQrVIqYQFnoECA8QAQ&usg=AOvVaw3g9cqnirAik2wM0XIlnxai)

### Chrony server configuration 

The trickiest part comes in configuring the time server. There is no separate utility in chrony to act as a time server, it is `chronyd` that acts as both client and server. Every chrony application is an NTP client unless it of stratum 0 (Calculates time independently using atomic clocks, GPS, etc). A little change in the configuration allows Chrony to serve time for clients. 

Directives that need to be added in the Chrony server configuration are given below:- 

```bash
           initstepslew 1 <client1> <client2> <client3>
           local stratum 8
           manual
           allow <subnet_in_CIDR_notation>
           smoothtime 400 0.01
           log measurements statistics tracking
```

The `initstepslew` allows the server to adjust the time on reboot by checking the time of its clients. The `local` directive is important as it makes the server appears to be of stratum 8 and is in sync with real time, eventhough it is not. Without specifying the stratum, the client considers the stratum of local time server to be higher than 15, thus making the server unreachable. 

The `manual` directive allows the time of the server to be set manually using `chronyc settime` . Once again, we are using the `allow` directive to permit the clients to request time. 

The `smoothtime` directive helps in adjusting the clock of the client when a large offset (difference) is observed between the server and client. 

## Tracking Chronyd

Chrony has a couple of components - Chronyd (daemon) and Chronyc (controller). We can use Chronyc to track and control the Chrony daemon, which is responsible for keeping time in sync.

The below command can be used to list the availble time servers and their status. 

```bash
chronyc sources -v 
```
If you get a '*' symbol next to your configured local time server, then you are in the right way. Chronyd is considering the local time server as the best choice. 

To check out which time server your client is syncing to right now, use the command below:- 

```bash
chronyc tracking
```
The value of 'Reference ID' is the time server from which your client is currently getting time. 

***I'm all green in time synchronization concepts. There is a ton of stuff to learn. So I always welcome constructive crticism. Although time is just a reference, an illusion, servers all over the world rely on them to sync, measure and coordinate. So let's better keep it in sync ;)***