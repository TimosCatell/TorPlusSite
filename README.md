# Setting up a site

## Cloud-based server
This is my experience setting up a site on a new service - [TorPlus](https://github.com/torplusdev/docs). The following is loosely based on [their docs](https://github.com/torplusdev/docs/blob/master/Docker%20Installation%20Instructions.md)

So, first, we need a domain name. If we want to stay _anonymous_, we'll need a crypto-friendly domain registration service (we'll also need BTC, ETH etc. and we'll want that to be anonymized as well. This is outside of the scope of this document). Some services that seem to support buying a domain name with crypto include [porkbun](porkbun.com) and [namecheap](https://www.namecheap.com/). Of course, if you want to keep yourself anonymous, you should use the TorPlus browser extension or the TorBrowser (which scrubs more identifiable details).

Next, we need a VPS. There are hundreds of options here, including several that support crypto payments. Check [this](https://bitcoin-vps.com/) list.  

After you provision a new host (I'm using Ubuntu below), use [TorSocks](https://gitweb.torproject.org/torsocks.git) or similar (TorPlus seems to expose SOCKS on port 29050. TorBrowser on 9150 and "vanilla" Tor on 9050).

```shell
% torsocks ssh root@vps
root@vps's password:
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-29-generic x86_64)

root@vps:~#
```

OK, we're in.

## Set up TLS

Next, let's set up TLS for our new domain.
We'll use [Let's Encrypt](https://letsencrypt.org/) as certificate authority.
They have a nice tool - `certbuddy` to help 
