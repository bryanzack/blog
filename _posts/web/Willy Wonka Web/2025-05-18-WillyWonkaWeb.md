---
layout: post
title: "Willy Wonka Web"
date: 2025-05-18
---



### About

This challenge requires the user to send a `GET` request to the root site page, and if the request includes the header key-value pair of `a: admin`, you will get the response as a flag.

`server.js`
```
// imports
const express = require('express');
const fs = require('fs');

// initializations
const app = express()
const FLAG = fs.readFileSync('flag.txt', { encoding: 'utf8', flag: 'r' }).trim()
const PORT = 3000

// endpoints
app.get('/', async (req, res) => {
    if (req.header('a') && req.header('a') === 'admin') {
        return res.send(FLAG);
    }
    return res.send('Hello '+req.query.name.replace("<","").replace(">","")+'!');
});

// start server
app.listen(PORT, async () => {
    console.log(`Listening on ${PORT}`)
});
```

### The Challenge
The apache2 server is configured to automatically remove all `a` or `A` headers. So the problem lies with having to bypass the `httpd.conf` settings to trick the backend into letting our `a: admin` header through to get the flag

`httpd.conf`
```
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

<VirtualHost *:80>

    ServerName localhost
    DocumentRoot /usr/local/apache2/htdocs

    RewriteEngine on
    RewriteRule "^/name/(.*)" "http://backend:3000/?name=$1" [P]
    ProxyPassReverse "/name/" "http://backend:3000/"

    RequestHeader unset A
    RequestHeader unset a

</VirtualHost> 
```


### CRLF Injection
We can inject HTTP Headers without proper sanitization through CRLF injection. CR+LF (\r\n) marks the end of an HTTP header line. By injecting these characters, we can add new headers **after** the initial header sanitization done by `httpd.conf` 

### The Payload
If we visit the following URL, we will receive the flag.

	https://wonka.chal.cyberjousting.com/name/%0d%0aa:%20admin%0d%0aX-Ignore:

This works because the upon initial processing, the payload `%0d%0aa:%20admin%0d%0aX-Ignore:` is not recognized as an actual header and avoids the `RequestHeader unset` lines in `httpd.conf`. The request is then rewritten and forwarded to the `/?name=` endpoint. This proxied request then looks something like this:

	GET /?name=[CR][LF]a: admin[CR][LF]X-Ignore: HTTP/1.1

Which is essentially the same as this:


​	
	GET /?name=
	a: admin
	X-Ignore HTTP/1.1


### The Flag
When the backend processes this request it then recognizes the `a: admin` and `X-Ignore:` as separate lines, therefore treating them as headers. With the proper header set we then receive the flag as a response.

![[firefox_vmEWTMK3DJ.png]]