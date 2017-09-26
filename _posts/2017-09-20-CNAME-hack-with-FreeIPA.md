---
title: CNAME hack with FreeIPA
updated: 2017-09-20 12:02
---


This work was a solution to a problem that came up in the subject's project System Administration 2. In that project I had to create two networks with different purposes. The first network had to be the servers network and the second the clients network. The servers network had to be composed by four servers:

- **FreeIPA master:** This server had to provided DNS, NTP, LDAP and Kerberos services integrated thanks to FreeIPA, in particular the [4.4.0](http://www.freeipa.org/page/Releases/4.4.0) version.
- **FreeIPA slave:** This server had to be a slave for failure cases of the master.
- **NFS server:** A server for storing the users homes.
- **Zabbix server:** A Zabbix server for monitoring the clients and the servers.

The clients network had to have five clients that consumed these services. Both servers and clients were CentOS 7.2 machines.


My teacher gave to each student a network with which had to create these sub-networks. In particular my network was `155.210.161.32/28`. So, I only needed one bit for dividing this network in two sub-networks. The sub-network `155.210.161.32/29` was asigned to the servers and `155.210.161.40/29` to the clients. The networks were connected as follows.

The servers sub-network was connected to an existing gateway (`155.210.161.33`) and had a second that routed the packets to the clients sub-network (`155.210.161.41`). The servers had the next directions:

- `155.210.161.35` was the FreeIPA master.
- `155.210.161.36` was the FreeIPA slave.
- `155.210.161.37` was the NFS server.
- `155.210.161.38` was the Zabbix server.


To understand this better you can see the next diagram, also, you can see the ip address assigned to each client (the names are in spanish, so, if you don't understand, please use a translator):

![Diagram](../postsImages/CNAME-hack-with-FreeIPA/network_diagram.png)


I configured the network and installed FreeIPA in the master server. Before install this, I changed the hostname to ns1.daniel.as2.unizar.es and I configured as forwarder the nameserver ns1.as2.uniar.es. In this server were defined already the necessary glue records to the future nameservers of the zone daniel.as2.unizar.es.

Well, the zone daniel.as2.uniar.es was defined automatically during FreeIPA installation, but FreeIPA installer couldn't find a reverse zone automatically, this is normal because in the server ns1.as2.unizar.es the reverse zone wasn't defined.

# The problem
And here comes the problem. As you have seen, my given network was `155.210.161.32/28`, this prefix is not multiple of 24, that is, a claseless ip. So, to solve this I found [this](https://www.freeipa.org/page/Howto/DNS_classless_IN-ADDR.ARPA_delegation) in the FreeIPA page. 

As you can read in the theory section, the clients look for names like w.x.y.z.in-addr.arpa because clients can't possibly know how networks are partitioned. The main trick is to use CNAME/DNAME records to redirect clients from w.x.y.z.in-addr.arpa to some other name. The target name can be arbitrary DNS name, i.e. the name can belong to different zone and normal delegation rules will apply.

The next section in the page is an example to understand how to solve it better. Well, this example is wrong and I will say why later. But first, look how the example defines the reverse zone with the syntax `a/w.x.y.z.in-addr.arpa.`. This will bring confusions in the future because we can mistake the prefix number (w) with the last number of the direction (a). There is another more simple and clear syntax.

Now I'm going to say why the FreeIPA's example is wrong: DNS names are reversed. It says that wants to delegate zone for classless network `198.51.100.0/26` in ipa1.example.com and it's in ipa2.example.com. At the same that the neartest class network is `198.51.100.0/24` and the entries are stored in ipa1.example.com and not in ipa2.example.com.

Ok, after all this explanations, I decided to do this better combining the solution of [FreeIPA page](https://www.freeipa.org/page/Howto/DNS_classless_IN-ADDR.ARPA_delegation) with the other syntax mentioned above. This work can be separated in two parts.

1. Define the glue records to an auxiliary zone.
2. Define the zone in the FreeIPA servers.

## Define the glue records to an auxiliary zone.




