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

## Set up a "workspace"
Now it's time to create a new user to work with (working as `root` is a bad idea):
```shell
root@vps:~# adduser tp
Adding user `tp' ...
Adding new group `tp' (1000) ...
Adding new user `tp' (1000) with group `tp' ...
Creating home directory `/home/tp' ...
Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for tp
Enter the new value, or press ENTER for the default
	Full Name []: 
	Room Number []:
	Work Phone []:
	Home Phone []:
	Other []:
Is the information correct? [Y/n] y
root@vps:~# usermod -aG sudo tp
root@vps:~# su - tp
tp@vps:~$
```

## Stellar
We need a Stellar wallet containing at least 1 LUX.
We can create a wallet directly on the new machine:

```shell
tp@vps:~$ sudo apt install npm
Reading package lists... Done
Building dependency tree
Reading state information... Done
npm is already the newest version (6.14.4+ds-1ubuntu2).
0 upgraded, 0 newly installed, 0 to remove and 255 not upgraded.
tp@vps:~$ sudo npm install -g stellar-wallet-cli
/usr/local/bin/stellar-wallet-cli -> /usr/local/lib/node_modules/stellar-wallet-cli/bin/cmd.js

> sodium-native@2.4.9 install /usr/local/lib/node_modules/stellar-wallet-cli/node_modules/sodium-native
> node-gyp-build "node preinstall.js" "node postinstall.js"

+ stellar-wallet-cli@1.2.1
added 67 packages from 116 contributors in 15.159s
tp@vps:~$ stellar-wallet-cli generate
-----------------------------------------------
Stellar Wallet Generate Wallet
-----------------------------------------------

  Public address: GASDQ3Q4YST53LK2QCX3PLFKPD4HIKWNBNWOFKVBZFLDVJETFRY5FLUA
  Wallet secret: SCDENN4TJQH5LF4CSXX4Y564EC43AEP47TLYUDXIP6ZX4EDRGT5B6QA6

  Print this wallet and make sure to store it somewhere safe!

  Note: You need to put at least 1 XLM on this key for it to be an active account

tp@vps:~$
```

As the output says, you will need to transfer at least 1 XLM to the new wallet to make it active (use the *public* address when transferring XLMs to the new wallet, of course).

You'll need to pass the secret to the TorPlus server you'll be starting soon.

## Docker
Next, let's install Docker, as TorPlus is pre-packaged as a bunch of Docker images.

``shell
tp@vps:~$ sudo apt install docker.io
Reading package lists... Done
Building dependency tree
Reading state information... Done
docker.io is already the newest version (20.10.12-0ubuntu2~20.04.1).
0 upgraded, 0 newly installed, 0 to remove and 254 not upgraded.
```
And let's pull the docker image we'll be using:
```shell
tp@vps:~$ sudo docker pull torplusdev/production:ipfs-latest
Digest: sha256:864c608fc647b42143f686237f7f34961ab043f36f815044d23f8cee9dac508d
Status: Image is up to date for torplusdev/production:ipfs-latest
docker.io/torplusdev/production:ipfs-latest
```

## DNS records
TorPlus uses DNS to resolve the onion address for a site. This is done by adding `torplus=<onion address>` to the TXT record (without the `.onion` part)

Here's how this looks for the `wikitpdemo.com` demo site:

```shell
tp@vps:~$ dig TXT torplus.wikitpdemo.com

...
;; ANSWER SECTION:
torplus.wikitpdemo.com.	1800	IN	TXT	"torplus=7v6hk5tugyq5g5wmnbsohasqicexbkhwzmtzu7ezy6nitvec42rs4iid"
...
```

You should be able to set this up with your DNS service provider.

You can find the onion address here:
```shell
tp@vps:~$ cat $workspace/hidden_service/hsv3/hostname
7v6hk5tugyq5g5wmnbsohasqicexbkhwzmtzu7ezy6nitvec42rs4iid.onion
```

(don't forget to remove the .onion part)

There's another small thing we should set up while we're here: we want https://torplus.mydomain.com to point to https://torplus.com/requires/.

Most domain services support "URL forwarding" which will handle such requests (without exposing your server's IP).

