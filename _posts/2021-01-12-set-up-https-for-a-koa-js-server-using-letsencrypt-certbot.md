---
layout: post
title: "Set Up HTTPS for a Koa.js Server Using LetsEncrypt (Certbot)"
categories: [node.js, ssl, javascript]
comments: true
---

The [Koa backend](https://github.com/graysonliu/spotify-charts-generator-server) of my small full-stack Node.js project has been upgraded to support HTTPS. Actually, since the frontend uses HTTPS, it cannot communicate with a server that only supports HTTP (browsers will block HTTP requests sent by a HTTPS page). I plan to use [LetsEncrypt](https://letsencrypt.org/) to acquire a free SSL certificate. Certbot, which LetsEncrypt recommends, has good support for popular servers like Apache and Nginx, but not with Node.js servers. You have to create the certificate manually for a Node.js server with certbot. A detailed [article](https://itnext.io/node-express-letsencrypt-generate-a-free-ssl-certificate-and-run-an-https-server-in-5-minutes-a730fbe528ca) written by David Mellul talks about how to use certbot to generate the certificate for Express servers. This article closely follows his instructions, except that Express is changed to Koa.

## Use Certbot to Create SSL Certificates

Create the certificate manually.

```bash
certbot certonly --manual
```

Enter your domain name:
```
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c' to cancel): <your-domain-name>
```

Give consent to IP logging. Then, the certbot will tell you what to do next on your server:

```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

<data>

And make it available on your web server at this URL:

http://<your-domain-name>/.well-known/acme-challenge/<name>

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Basically, you have to serve a static file on your server. The file is in directory `.well-known/acme-challenge` and it is named `<name>`. The content of the file should be `<data>`.

I use `koa-static` to server static files. Since my server is an API server, this certbot file is the only thing that my server will treat as static resources. I created a dedicated directory named `letsencrypt` to serve this static file. Put file `<name>` into directory `./letsencrypt/.well-known/acme-challenge`, and use `koa-static` as follows:

```javascript
const static = require('koa-static');

app.use(static('./letsencrypt', {hidden: true})); // serve static files
app.listen(80);
```

Note that `.well-known` is a hidden directory. We should set option `hidden` to `true` to serve hidden files/directories as static resources. Also, your server most likely does not use the default HTTP port `80`. Make your server listen to port `80` for now. You can change it back to whatever you like after.

When all above have been done, visit that URL in your browser to check whether the setup is correct, and you should see the content `<data>` displayed. Then, back to your terminal and just follow the instructions that certbot gives you.

## Use Certificates in the Koa Server

Certbot put created keys in `/etc/letsencrypt/live/<your-domain-name>` by default. In Koa, use the following code (I use port `3000`):

```javascript
const fs = require('fs');

// SSL for HTTPS
const options = {
    key: fs.readFileSync('/etc/letsencrypt/live/<your-domain-name>/privkey.pem'),
    cert: fs.readFileSync('/etc/letsencrypt/live/<your-domain-name>/cert.pem'),
    ca: fs.readFileSync('/etc/letsencrypt/live/<your-domain-name>/chain.pem')
};

https.createServer(options, app.callback()).listen(3000);
```

Now, your server provides services using HTTPS.

## Use cron and nodemon for Automatically Renewal

It should be noted that the certificates issued by LetsEncrypt will expire in 90 days. You have to renew certificates periodically. To renew manually created certificates, use

```bash
certbot renew --standalone
```

This command will only renew certificates that expires in 30 days. You can force the renewal by using

```bash
certbot renew --standalone --force-renewal
```

However, LetsEncrypt has rate limits. Forced renewal might fail if you exceed the limit.

I also use cron to schedule the renewal monthly. First, create a bash script file named `ssl_renew.sh` in your project directory. Make this file executable and write command `certbot renew --standalone` in it. Use `crontab -e` and append the following line in the opened editor:

```bash
0 0 1 * * /<path-to-your-project>/ssl_renew.sh
```

This basically renews certificates on the first day of every month. After the renewal, your server should be restarted to use the new certificates. This can also be done automatically. I use the tool [nodemon](https://www.npmjs.com/package/nodemon) to achieve that. nodemon can automatically restart the node application when file changes in directories are detected. To use nodemon, replace `node` with `nodemon` in `package.json` when executing your script:

```json
"scripts": {
    "start": "nodemon app.js"
  }
```

Also, add configurations for nodemon in `package.json`:

```json
"nodemonConfig": {
    "ignore": [
      ".git",
      "test.js"
    ],
    "delay": "2500",
    "watch": [
      ".",
      "/etc/letsencrypt/live/<your-domain-name>"
    ],
    "ext": "js, json, pem"
  }
```

There are two related attributes in this configuration. First, `watch` indicates directories or files where nodemon should monitor changes. Obviously, we should add `/etc/letsencrypt/live/<your-domain-name>`, where the certificates are saved. Also, by default, nodemon only detects changes in files with some specific extensions like `.js` and `.json`. In attribute `ext`, we should also add `pem`, which is the extension of certificates. Now, after the renewal, the Node.js server will automatically be restarted by nodemon.

Check my [project](https://github.com/graysonliu/spotify-charts-generator-server) for more details.



