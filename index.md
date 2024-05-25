# I recommend you to check "localhost" in nginx right now.
We use nginx on our server. At the beginning of the introduction, web server "localhost" was set on nginx for use in operation check etc.
After completing the DNS settings and the test / production WEB server settings, there is no need to check the operation and access to "localhost" is no longer necessary.
However, since it is expected that operation confirmation will be required in the future, the "localhost" setting was left as it is.
As a side note, **The root folder of "localhost" was set to use the same location as the production web server (www.[ my domain ])**。
## An ERROR has occured
A suspicious error occurred while checking the logs. Our website uses Google Firebase Analytics. The API key used here is restricted by the HTTP referrer, but this usage restriction violation causes an error.
Tracking the logs revealed that this error was occurring on access from "localhost". Also, the client is Linux. On our server, only infrastructure personnel can log in to the server, not application personnel. This error is not possible. I thought I was hacked, so I was a little impatient.
After further tracking the source of this error, I came to the conclusion that it was a web crawler. [[Official document: How Google Search Works (for beginners)]](https://developers.google.com/search/docs/beginner/how-search-works#crawling) The web crawler was using the "localhost" web server by accessing `hssp: //xxx.xxx.xxx.xxx`.

1. It's not being hacked by hackers, and there are no security issues. They're just looking at pages that are open to the public
1. There is an error, but it is not an error that is occurring for general users
1. I can't get Analytics data, but it's convenient because I want to exclude the number of web crawler accesses from the count.

I decided that it wasn't really harmful and left the error as it was.**This was the root of all the problems**
##Problem outbreak
Recently, when I searched on Google, our website was a hit. This is a delight in itself, but **the address linked there was `http://xxx.xxx.xxx.xxx/~`!**
The web crawler was working properly.
It became a troublesome situation

1. Even though the production address is prepared by https, it is accessed by http
1. The server is to be changed. It becomes a problem when fixed addresses spread to the world
1. I want to avoid errors during use by general users. It is very troublesome for general users to use "localhost" because it is premised on the usage restrictions in the HTTP referrer.

I have to recover somehow ...
##Workaround
###reverse proxy
I set reverse proxy in "localhost" on nginx. I tried to redirect the address to the www.[ my domain ].
The API key HTTP referrer error has been resolved, but this remains a problem

1. It works fine with **http://~**, but there are other issues with **https://~**. Because no one can get **CA signed certificate** for **"localhost"**. Since it should be a **self-signed certificate**, I get a "connection is not private" error in Chrome.
1. When someone visits `http: // xxx.xxx.xxx.xxx / ~`, the website returns the correct page.I'm worried that web crawlers will continue to recognize it as a legitimate site. Then, this link as a search result will remain displayed. It will be a problem when we move the server in the future.

I have to recover somehow ...
###Hello World
I changed our website to show Hello World when someone visits `http://xxx.xxx.xxx.xxx/~`
In this case, "localhost" can be used as an operation check. Since it is a meaningless page, I expect that the web crawler will eventually exclude it from the search results. 
But this is a problem with this

1. General users who access from the search results will be surprised. "What is this?"
1. Even if you access it, "Hello World" will be returned with normal status, so the web crawler may not recognize it as a bad site to access.

I have to recover somehow ...
###Show a customized 404 error screen
In conclusion,  I set our nginx to show custom 404 error screen when someone visits `http://xxx.xxx.xxx.xxx/~` or `https://xxx.xxx.xxx.xxx/~`.
It is like this.

```
An error occurred.
Sorry, you can not use the IP address for the page you are looking for.
Please try domain name like "https://www.[ my domain ]/~".

Faithfully yours.
```
For reference, the settings of nginx to display the custom error page are as follows.

```server.conf
server {
    listen       443  ssl;
    access_log  /var/log/nginx/localhost.access.log  main;
    error_log  /var/log/nginx/localhost.error.log;
    ssl_certificate /etc/nginx/ssl/nginx.pem;      # this is self-signed certificate
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    server_name  localhost;    
    error_page 404 /404.html;                      # custom error page name
    location / {
        return 404;                                # allways occur 404 error
    }
    location = /404.html {                         
        root   /usr/share/nginx/localhost;         # 404.html is saved here
        internal;
    }
}
server {
    listen       80;
    server_name  localhost;
    access_log  /var/log/nginx/localhost.access.log  main;
    error_log  /var/log/nginx/localhost.error.log;
    error_page 404 /404.html;
    location / {
        return 404;
    }
    location = /404.html {
        root   /usr/share/nginx/localhost;
        index  index.html index.htm;
    }
}
```

Since the custom error screen is displayed,

1. In the case of operation check and connection test, we can check whether it worked correctly
1. It's similar to nginx original 50x.html, so it may be easily accepted by the users.
1. Since the reason for the error and the remedy are described, general users should not be surprised.
1. Since it returns with an error status, the web crawler may know that it is a bad page to access.

I decided to wait for the web crawler to correct the search results.

####Appendix
I regret that I should have set it like this from the beginning. If anyone has set up the "localhost2 WEB server with the same content as the production, I recommend that you change it now.

## Plese visit our website.
### [WEB System Infrastructure Guide for Beginners](https://olto3-sugi3.tk/index.html)
### [Easy Web Archiver](https://olto3-sugi3.tk/sticky-note/index.html)

