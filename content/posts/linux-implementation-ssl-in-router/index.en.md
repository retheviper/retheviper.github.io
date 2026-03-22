---
title: "Insert SSL certificate into router"
date: 2019-12-01
translationKey: "posts/linux-implementation-ssl-in-router"
categories: 
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - ssl
  - linux
---
The router I use at my parents' house is Asus' RT-AC58U. The one I personally use is RT-AC68U, also from ASUS. These two differ not only in machine specs but also in the functions provided at the firmware level. For example, functions such as AiMesh are not supported by RT-AC58U.

Then, when I updated the firmware the other day, I was restricted to only being able to access the management page from WAN via https. In the case of RT-AC68U, there is no particular problem as it has the function to create an SSL certificate [^1] with [Let's encrypt](https://letsencrypt.org) and update it automatically, but unfortunately this is not the case with RT-AC58U. So whenever I connect to the RT-AC58U management page from a browser, I get angry that the certificate is incorrect.

I think there is a possibility that it will be fixed in the future, but recently new models compatible with [802.11ax](https://en.wikipedia.org/wiki/IEEE_802.11ax) have been released one after another, so it is unclear how long firmware updates for the already outdated RT-AC58U will continue. And I get a little worried when I see people getting annoyed every time the certificate is incorrect.

I thought that the software itself would not be that different between the routers at my parents' home and the router at home, and when I looked into it, it turned out to be the same, so I thought there might be a way to manually insert the SSL certificate. When I investigated further, I found that the OS was Linux, and the conclusion was that I was able to successfully add the certificate to the 58U.

This time, I will explain how to apply the SSL certificate to RT-AC58U.

## System configuration

Here is a pictorial representation of the current system configuration.

![Configuration diagram](ssl_organization.webp)

What I want to do here is to put an SSL certificate into the router that registered DDNS[^2] so that I don't get scolded on the management page connected via https. Another reason why I tried this is because I also want to install a Linux machine that will function as a home server under this router later. I plan to install and operate a simple web application on my home server later, and if what I tried this time is successful, I think I can apply the SSL certificate there as well using the same mechanism.

Now, I will explain how I created an SSL certificate, uploaded it to the router, and applied it.

## Router settings (1)First, you need to configure DDNS settings on your router. For ASUS routers, you can access the local router management page by entering `http://router.asus.com` from a browser such as Chrome. Then select "WAN" from the "Advanced Settings" menu, then go to the DDNS tab and register it as your preferred address. ASUS routers provide a free server called asuscomm.com, so use that. Once you have registered DDNS, set "Allow connections from WAN" to "Yes" on the "System" tab under the "Management" menu. In order to connect from home, I had set up DDNS on my router at home in advance.

After configuring the management page connection settings for DDNS, the next step is to configure the SSH connection to the router. This can also be set from the "Management" page. After setting up the SSH connection, test it, make sure that you can access it using a public key, and change the port number. If you change the SSH port, you can access it with the following command in the terminal.

```bash
# If the SSH port is 2022
$ ssh -p 2022 retheviper@javaman.asuscomm.com
```

The ID and address for SSH will be the ID on the management page and the one registered in DDNS. Once this is done, the preparations on the router side for creating an SSL certificate are complete.

## Settings on mac(1)

The router's OS is Linux, but some important commands are still missing. It doesn't have `yum`, `apt`, or `dnf`, which are typically used for package management, and the CPU performance is questionable, so I decided to do important work on a Mac first.

Also, the SSL certificate itself uses Let's encrypt, which is supported by RT-AC68U. This is only valid for 90 days, but issuance and renewal are free, making it ideal for simple tasks like this.

First, install Let's encrypt in your terminal.

```bash
brew install letsencrypt
```

Once installed, you can create a certificate using the `certbot` command. However, before creating the certificate, you need to register and install DDNS. I have already registered the domain using the functionality provided by the router, so I will use it as is.

```bash
sudo certbot certonly --manual
```

When you enter the command, a screen like the one below will be output. However, since I have run the same command several times, the screen output when I run it for the first time may be slightly different.

```bash
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): [domain]
```

Enter the DDNS domain used by the router and press Enter to move to the next screen.

```bash
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for [domain]

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
```

You will be asked if you agree to have your IP recorded. I have no choice but to agree, so enter `Y`. Then the following screen will appear.

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

[code]

And make it available on your web server at this URL:

[http address]

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue  
```

Stop at this screen and copy the code and URL displayed on the screen. I'll come back here later.

## Settings on PCThe screen that was output earlier means "We will send a request to this URL, so please make sure that we can get this code as a response." Therefore, it is necessary to temporarily set up a server so that it can respond.

There is a way to simply create a file that can be accessed on the server, but I decided to try a different method. All you need to prepare is to set up a server that can provide responses on the PC connected to the router. You can do it with a router if its performance is sufficient, but my RT-AC58U died for a while just by downloading Python and unzipping the compressed file. Here, I will create a simple server using Node.js on my PC. You can also use Python, Ruby, etc. This is just because Node.js was the fastest way for me to set up a server.

My PC at home runs Windows, so I downloaded Node.js from the [official homepage](https://nodejs.org) and installed it. We will also use express to build a server. Once the installation is complete, you will be able to use npm from the command line. You can create an express starter project with the code below.

```cmd
> mkdir node
> cd node
> npm install express
```

After this, use a text editor such as VSCode to create the code below. Name the file `app.js` and save it in the folder where you installed express earlier. Don't forget to enter the URL and code you copied earlier.

```javascript
var express = require('express')
  , http = require('http')
  , app = express()
  , server = http.createServer(app);

app.get('/[copied http address]', function (req, res) {
    res.send('copied code');
  });

server.listen(80, function() {
  console.log('Express server listening on port ' + server.address().port);
});
```

Save the file and run it from the command line to start the server. It can be executed with the following command.

```cmd
> node app.js
```

Once the server has started, check if you can access it locally. Enter the URL in your browser and confirm that the code is displayed correctly, then the settings on your PC are complete.

## Settings on mac(2)

With the server running on your PC, return to your Mac. When you press enter, communication with the server will begin, and the following screen will be output as a result.

```bash
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/javaman.asuscomm.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/javaman.asuscomm.com/privkey.pem
   Your cert will expire on 2020-02-14. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

The creation of the SSL certificate has been successfully completed. From this screen, you can see where the certificate is saved and its expiration date. Also, if you enter cerbot renew, it will tell you that you can renew.

Now that you have created an SSL certificate, all you need to do is copy it to your router and apply it. First, enter the path displayed on the screen and copy the following files.

-cert.pem
-key.pem

Once the copy is complete, connect it to your router.

## Settings on router(2)Connect to the router with SSH and navigate to the path below.

```bash
cd /tmp/etc
```

If you look at the files in the directory, you can see that they contain the same files that you copied earlier. Open the file with vi and overwrite it with the one you copied earlier.

After overwriting cert.pem and key.pem, check the process list inside the router. Since the management page is already running as an httpds service, you will need to stop and restart the service to apply the new certificate.

```bash
ps
```

Entering the above command will output a list of currently running processes. If there is `httpds -s -i br0 -p 8443`, it will be terminated. 8443 is the default port number specified on the admin page. Remember that the process ID (PID) is output to the left of the process. Then enter the following command:

```bash
# If the PID is 562
$ kill 562

# Restart the process
$ /usr/sbin/httpds -s -i br0 -p 8443 &
```

Please note that you will not be able to enter any other commands unless you enter `&`. Once the input is complete, enter `ps` again and if the process starts properly, the settings here are complete. After pressing `exit` to exit from ssh, connect to the router's management page from the browser and check if it prompts you for the certificate. If there were no particular problems during the process so far, there should be no problem.

The only thing you need to be careful about is restarting your router. I try to restart it at least once a week, but in this case, it seems that the SSL certificate value I put in is initialized. Therefore, it is necessary to avoid rebooting as much as possible, or to re-enter the certificate after rebooting.

## Finally

If there are no particular problems with the above, you will no longer get angry when you access the router's management page from the WAN because the certificate is incorrect. Now you can manage it from outside with peace of mind!

However, this doesn't mean everything is perfect. The remaining tasks are below.

- How to renew the certificate
- How to deal with router restarts

The certificate created by Let's encrypt is valid for 90 days, so you will need to renew it later. The update itself can be easily completed by typing the certbot command, but post-update processing (uploading to the router, restarting the application) is required. This seems to work if you schedule it with `crontab`, but unfortunately it wasn't included as a command in the router.

It is subject to verification how it will respond when the router restarts. At first, I thought that I could solve this by creating a shell script that overwrites the file with scp and also restarts the httpsd process, but there may be permissions issues, so I'm wondering if there is an easier way to do it.Well, if you end up building a server with Linux, it's possible that you won't have to connect to the router's management page from the WAN. Anyway, if I find out anything, I'll write another post.

So, everyone, please enjoy a safe and comfortable web life with an SSL certificate!

[^1]: An SSL certificate is an electronic document that proves that this server can be trusted. By applying an SSL certificate, communication via https is protected from attacks by third parties.
[^2]: Abbreviation for Dynamic Domain Name System. Home routers often have IP addresses that change dynamically, but this is a convenient service that connects this with a string host name. Regardless of the router's IP address, if you have configured DDNS, you can always access the router from the same URL.
