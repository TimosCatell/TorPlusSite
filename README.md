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
tp@vps:~$ mkdir work
tp@vps:~$
```

## TLS/SSL
We will need a valid TLS certificate for our site.
You can use [Let's Encrypt](https://letsencrypt.org/) to generate a free certificate. There's even a nice [tool](https://certbot.eff.org/) that makes this process effortless.

There's just one problem: your IP will be logged:
```shell
root@vps:~# certbot certonly --manual --preferred-challenges=dns -d abc.xyz
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for abc.xyz

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o:
```

This may or may not be a problem.
Note that some name service providers (e.g. porkbun) will generate Let's Encrypt certificates automatically. You can then download these certificates to the VPS - which does not involve giving up your IP address (the downside is you'll need to refresh the certificate manually every 3 months).

Assuming you have downloaded the certificate, you should now have the "full-chain" certificate, and private key. Assuming `domain.cert.pem` is the certificate and `private.key.pem` is the private key, we'll need to concatenate them and put the resulting file in the `ssl` directory:

```shell
tp@vps:~$ export domain=abc.xyz
tp@vps:~$ mkdir -p work/ssl/
tp@vps:~$ cat domain.cert.pem private.key.pem > work/ssl/$domain.pem
```

with `abc.xyz` replaced with your domain name, of course.


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

```shell
tp@vps:~$ sudo apt install docker.io
Reading package lists... Done
Building dependency tree
Reading state information... Done
docker.io is already the newest version (20.10.12-0ubuntu2~20.04.1).
0 upgraded, 0 newly installed, 0 to remove and 254 not upgraded.
```

And let's pull the docker image we'll be using:
```shell
tp@vps:~$ sudo docker pull torplusdev/production:ipfs_haproxy-latest
ipfs_haproxy-latest: Pulling from torplusdev/production
Digest: sha256:f577b5eb87f44884122df1e2b966b22e60934cf49eb2361ef9446adfff6dbcd4
Status: Image is up to date for torplusdev/production:ipfs_haproxy-latest
docker.io/torplusdev/production:ipfs_haproxy-latest
```

Start the image:
```shell
tp@vps:~$ sudo docker run -d --name torplus \
-e nickname=tpvps \
-e seed=SCDENN4TJQH5LF4CSXX4Y564EC43AEP47TLYUDXIP6ZX4EDRGT5B6QA6 \
-e role=hs_client \
-e HOST_PORT=80 \
-e PP_ENV=prod \
-e http_address=127.0.0.1:80 \
-e useNginx=1 \
-p 127.0.0.1:80:80 \
-p 127.0.0.1:28000:28080 \
-p  127.0.0.1:5001:5001 \
-v /home/tp/work/tor:/root/tor \
-v /home/tp/work/ipfs:/root/.ipfs \
-v /home/tp/work/hidden_service:/root/hidden_service \
-v /home/tp/work/static:/var/www/html \
-v /home/tp/work/ssl:/etc/ssl/torplus/ \
--rm torplusdev/production:ipfs_haproxy-latest
2389b2f3dafa34bba3f6571525a59085f3fcce81e36ed2ba0d6e77133e61505f
tp@vps:~$ 
```

Where you should replace the seed (`SC...`) with your Stellar secret.

Note that you can access http://127.0.0.1:28000 to check the status of the server. Similarly http://127.0.0.1:80 will access the site, without going through TorPlus. Needless to say, don't expose those the the big bad Internet: use ssh tunneling or something similar to forward these ports.

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

Don't forget to remove the .onion part! and note that the TXT record is for *torplus*.abc.xyz.

There's another small thing we should set up while we're here: we want https://torplus.mydomain.com to point to https://torplus.com/requires/.

Most domain services support "URL forwarding" which will handle such requests (without exposing your server's IP).

## Create a minimal site
We can now create a minimal site that will play a video.

First, let's install IPFS on the host machine (we only need the IPFS CLI):

```shell
tp@vps:~$ sudo snap install ipfs
```

(you'll be prompted if you need to install snap)

Let's see that we can connect to the TorPlus IPFS running in docker:
```shell
tp@vps:~$ ipfs --api /ip4/127.0.0.1/tcp/5001 stats bw
Bandwidth
TotalIn: 101 kB
TotalOut: 19 kB
RateIn: 4.1 kB/s
RateOut: 1.5 kB/s
```

### Upload to IPFS
Assuming we have a video file:
```shell
tp@vps:~$ ls -l video.mp4
-rw-r--r-- 1 tp tp 140152709 Jul 20 11:23 video.mp4
```

Let's upload it into the local IPFS node.

```shell
tp@vps:~$ ipfs --api /ip4/127.0.0.1/tcp/5001 add video.mp4
added QmPmhva4fcVs8P5iTT2psjqQGMGrkXmXTewNkHsR6n5gUY video.mp4
 133.66 MiB / 133.66 MiB [==================================================================] 
tp@vps:~$ 
```

We've added `QmPmhva4fcVs8P5iTT2psjqQGMGrkXmXTewNkHsR6n5gUY` to the local IPFS node. We can now create a mini-site that will simply play this file (note that although initially, the file will be served by our server, other nodes will ultimately cache the video and serve it to clients without hitting our server. At least, that's TorPlus claims on their site).

### Create a minimal site
Now let's create the site.
We'll use the sample HTML code from the TorPlus GitHub page:
```shell
tp@vps:~$ cat work/static/index.html
<html><head><meta charset="utf-8"><title>video</title></head>
<body>
   <video width="400" height="300" controls="controls" controlsList="nodownload" autoplay muted>
      <source src="/ipfs/QmPmhva4fcVs8P5iTT2psjqQGMGrkXmXTewNkHsR6n5gUY">
   </video>
</body>
</html>
```

### Check that site works
Point your TorPlus-enabled Chrome browser at your side - e.g. https://torplus.abc.xyz and you should see the video.


