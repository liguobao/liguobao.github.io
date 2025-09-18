---
layout: post
title: Jexus Supports HTTPS Protocol
category: jexus
date: 2016-10-04
tags:
- dotnet core
---

As we all know, when HTTPS pages request HTTP resources, modern browsers will intercept, prompt the user whether to continue, or directly intercept without prompting.

Recently, I made a quick bookmark tool for myself. Clicking the bookmark directly sends the bookmark to the server address and saves it to my website.

At first, everything was normal, but when I encountered HTTPS websites, it crashed.

At first, I saw that HTTPS certificates are charged, so I thought forget it, as long as it works. A few days ago, I occasionally saw an open source software for free HTTPS application, and it looked good. These days I had time and started working on it. Below is a tutorial, and I applied for the certificate almost according to this.

[Get Free Certificate with Let's Encrypt](https://www.paulyang.cn/blog/archives/39?spm=5176.blog2666.yqblogcon1.12.Nu0TgL)

About this Let's Encrypt, Wikipedia introduces it like this:

> Let's Encrypt is a digital certificate certification authority that will be launched at the end of 2015, which will provide free SSL/TLS certificates for secure websites through an automated process designed to eliminate the complexity of manually creating and installing certificates. Let's Encrypt is a service provided by the Internet Security Research Group (ISRG, a public welfare organization). Major sponsors include the Electronic Frontier Foundation, Mozilla Foundation, Akamai, and Cisco. On April 9, 2015, ISRG and the Linux Foundation announced cooperation to implement the protocol for this new digital certificate certification authority called Automatic Certificate Management Environment (ACME). There is a draft of this specification on GitHub, and a version of the proposal has been published as an Internet draft. Let's Encrypt claims that this process will be very simple, automated, and free. On August 7, 2015, the service updated its launch plan, expecting to release the first certificate on September 7, 2015, and then issue a small number of certificates to whitelisted domains and gradually expand. If everything goes as planned, the service is expected to fully start providing on November 16, 2015.

The entire project has code on Github, mainly through the client to generate https certificates for our website.

First, we download the client, as follows:

```shell
git clone https://github.com/letsencrypt/letsencrypt.git
```

Then enter this repository and execute the following code:

```shell
./letsencrypt-auto certonly -a webroot --webroot-path website path (such as: /var/www/web/) -d your domain (such as: test.online) -d www.your domain (such as: www.test.online)
```

Here, note that I added line breaks for typesetting here, remember to remove the line breaks when running this command.

Line breaks are in front of webroot and -d.

If everything goes well, we can see four files in the `/etc/letsencrypt/live/domain/` directory, namely:

1. Domain certificate file

2. Certificate chain file that issued the domain certificate

3. Domain certificate + certificate chain file

4. Private key file

As shown in the figure:

![letsencrypt file](http://7xread.com1.z0.glb.clouddn.com/60e4f29a-6da5-40e1-ae32-453a3bbf2455)

Then set the certificate for the website.

Jexus setting HTTPS requires changing the jws.conf document and the website configuration document.

Operation steps are as follows:

1. Modify jws.conf

Enter the Jexus folder, open "jws.conf", add the following two sentences:

```shell
CertificateFile    = /etc/letsencrypt/live/domain/fullchain.pem
CertificateKeyFile = /etc/letsencrypt/live/domain/privkey.pem
```

The effect after modification is as follows:

![Image description](http://7xread.com1.z0.glb.clouddn.com/d306d9c5-6391-421d-86fc-053b97d1b489)

2. Enable HTTPS for the website

Enter the siteconf/ folder, find the corresponding website conf file,

Change the website service port to 443:

port=443

Enable https:

UseHttps=true

The effect after modification is as follows:

![Image description](http://7xread.com1.z0.glb.clouddn.com/0800dc87-2500-42d2-a3c5-a75a2c819330)

Then restart jexus.

After completion, you can access through HTTPS.

Finally, upload a picture of the HTTPS certificate to prove that this is feasible.

![Image description](http://7xread.com1.z0.glb.clouddn.com/24842774-311e-4e55-a6b5-b88a89edc754)

Sprinkle flowers, see you next time.
