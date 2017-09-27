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


I configured the network and installed FreeIPA in the master server. Before install this, I changed the hostname to `ipa41.daniel.as2.unizar.es` and I configured as forwarder the nameserver `ns1.as2.uniar.es`. In this server were defined already the necessary glue records to the future nameservers of the zone `daniel.as2.unizar.es`.

Well, the zone `daniel.as2.uniar.es` was defined automatically during FreeIPA installation, but FreeIPA installer couldn't find a reverse zone automatically, this is normal because in the server `ns1.as2.unizar.es` the reverse zone wasn't defined.

# The problem
And here comes the problem. As you have seen, my given network was `155.210.161.32/28`, this prefix is not multiple of 24, that is, a claseless ip. So, to solve this I found [this](https://www.freeipa.org/page/Howto/DNS_classless_IN-ADDR.ARPA_delegation) in the FreeIPA page. 

As you can read in the theory section, in reverse DNS resolution, the clients look for names like w.x.y.z.in-addr.arpa because clients can't possibly know how networks are partitioned. The main trick is to use CNAME/DNAME records to redirect clients from w.x.y.z.in-addr.arpa to some other name. The target name can be arbitrary DNS name, i.e. the name can belong to different zone and normal delegation rules will apply.

The next section in the page is an example to understand how to solve it better. Well, this example is wrong and I will say why later. But first, look how the example defines the reverse zone with the syntax `a/w.x.y.z.in-addr.arpa.` This will bring confusions in the future because we can mistake the prefix number (w) with the last number of the direction (a). There is another more simple and clear syntax.

Now I'm going to say why the FreeIPA's example is wrong: DNS names are reversed. It says that wants to delegate zone for classless network `198.51.100.0/26` in `ipa1.example.com` and it's in `ipa2.example.com`. At the same that the neartest class network is `198.51.100.0/24` and the entries are stored in `ipa1.example.com.` and not in `ipa2.example.com.`

Ok, after all this explanations, I decided to do this better combining the solution of [FreeIPA page](https://www.freeipa.org/page/Howto/DNS_classless_IN-ADDR.ARPA_delegation) with the other syntax mentioned above. This work can be separated in two parts.

1. Define the glue records to an auxiliary zone.
2. Define the auxiliary zone in the FreeIPA servers.

## Define the glue records to an auxiliary zone
In this part I'm going to delegate the zone `155.210.161.32-47` to the servers `ipa41.daniel.as2.unizar.es` and `ipa42.daniel.as2.unizar.es.` To get this, into the file `named.conf` on the nameserver `ns1.as2.unizar.es` that contains the entries for the zone `155.210.161.32/24`, or the same `155.210.161.0-255`, I added the next content:

```
; alumno daniel rueda
$GENERATE 32-47 $ CNAME $ .32-47
35 CNAME 35.32-47
36 CNAME 36.32-47
32-47.161.210.155.in-addr.arpa. NS ipa41.daniel.as2.unizar.es.
32-47.161.210.155. in-addr.arpa. NS ipa42.daniel.as2.unizar.es.
``` 


The words that start with `$` are DNS file commands. With `$GENERATE` command, it generates a set of entries, specifically since 32 to 47 and saves each entry in `$`. `CNAME` gives a new name, in this case `$.32-47`, that is, the entry followed by `.32-47`. For example, the new name of entry 38 will be `38.32-47`. To understand better all this explanation, you can read [this part](https://books.google.com.hk/books?id=GB_O89fnz_sC&lpg=PA406&dq=bind%20cname%20hack%20byte%20boundary&hl=es&pg=PA400#v=onepage&q=bind%20cname%20hack%20byte%20boundary&f=false) of the book Linux Administration Handbook. Finally, in the last two lines it defines the authority servers of the zone `32-47.161.210.155.in-addr.arpa.` All this is better to understad with an example.

### Example
Let's supose that some machine will search which is the name of the ip `155.210.161.38`, that is, it will search the entry `38.161.210.155.in-addr.arpa.` As we know, reverse DNS search from right to left. It will have found the zone `161.210.155.in-addr.arpa.` in the server `ns1.as2.unizar.es.` The next entry to find is `38.`. The resolution finds that `38.` has a new name which is `38.32-47.` Then, it has to access to the zone `32-47.161.210.155.in-addr.arpa.` and it finds that the authority nameservers are `ipa41.daniel.as2.unizar.es.` and `ipa42.daniel.as2.unizar.es.` (my servers). By last, the resolution will find the given ip in `ipa41.daniel.as2.unizar.es.` or in `ipa42.daniel.as2.unizar.es.`

## Define the auxiliray zone in FreeIPA servers
Finally, it's time to define the auxiliary zone in the authority servers, now in FreeIPA master (`ipa41.daniel.as2.unizar.es`) I added the auxiliary zone.

```
[ a559207@ipa41 ~] $ ipa dnszone-add 32-47.161.210.155.in-addr.arpa.
Zone name : 32-47.161.210.155.in-addr.arpa.
Active zone : TRUE
Authoritative nameserver : ipa41.daniel.as2.unizar.es.
Administrator e-mail address : hostmaster
SOA serial : 1497869513
SOA refresh : 3600
SOA retry : 900
SOA expire : 1209600
SOA minimum : 3600
BIND update policy : grant DANIEL.AS2.UNIZAR.ES krb5-subdomain 32-47.161.210.155.in-addr.arpa. PTR
Dynamic update : FALSE
Allow query : any ;
Allow transfer : none ;
``` 

After this, I added the `PTR` entries:

```
[ a559207@ipa41 ~] $ ipa dnsrecord-add 32-47.161.210.155.in-addr.arpa. 35 --ptr-rec ipa41.daniel.as2.unizar.es.
  Record name : 35
  PTR record : ipa41.daniel.as2.unizar.es.
[ a559207@ipa41 ~] $ ipa dnsrecord-add 32-47.161.210.155.in-addr.arpa. 36 --ptr-rec ipa42.daniel.as2.unizar.es.
  Record name : 36
  PTR record : ipa42.daniel.as2.unizar.es.
[ a559207@ipa41 ~] $ ipa dnsrecord-add 32-47.161.210.155.in-addr.arpa. 37 --ptr-rec nfs4.daniel.as2.unizar.es.
  Record name : 37
  PTR record : nfs4.daniel.as2.unizar.es.
[ a559207@ipa41 ~] $ ipa dnsrecord-add 32-47.161.210.155.in-addr.arpa. 38 --ptr-rec zabbix1.daniel.as2.unizar.es.
  Record name : 38
  PTR record : zabbix1.daniel.as2.unizar.es.
```

# Conclusion
As you can see, I've made a new and simply form of CNAME hack with FreeIPA, with a clearest syntax.