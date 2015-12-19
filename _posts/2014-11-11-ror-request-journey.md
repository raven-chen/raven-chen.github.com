---
layout: post
title: "Journey of the request that posted to a Ruby on rails website"
description: "Setup Ruby on rails production environment"
category:
tags: [rails, server, deploy]
---


I've deployed a small Ruby on rails project(Nginx + Unicorn + Mysql) on an initial cloud host(CentOS). By this practice. I studied and understood the whole workflow of how server process a request. I was about to summary all my works into a blog that list every steps like 1. Setup Nginx 2. Setup Unicorn ... But when I look at my outline I realized that if I express my experience in another perspective then it will be more clear.

## The workflow

1. User type domain name in browser and press enter.
2. DNS server parsing user's request and map it to server(IP)
3. Nginx received request and pass it to Unicorn
4. Unicorn process the request by logic defined in project(talk to Mysql if needed) and send response to Nginx
5. Nginx received response from unicorn and send it back to browser
6. Browser render response to user

## Detailed workflow

### 1. DNS

Need a domain name provider to tell DNS servers the mapping between your domain name and your server's IP address so user(Browser/Application) know where to send user's request. Basically you have to tell everybody on the web that domain `www.example.com` point to my server `x.x.x.x`.

### 2. Nginx

Brief Nginx config about how to connect Nginx and Unicorn.

<pre>
server {
  listen 80;
  server_name www.example.com; # Listen request to your website domain
}

upstream demo_app {
  server unix:/tmp/unicorn.sock;
  # server 55.192.40.236:8080; if you web server and app server on different hosts
}

location @demo_app {
  proxy_pass http://demo_app; # Nginx will pass request to unicorn by this and upstream set
}
</pre>

### 3. Unicorn

Unicorn has a sample `unicorn.conf` file. Three important variables.

1. working_directory # The place you can run `rails s`
2. unicorn\_sock_file # The bridge between Nginx and Unicorn
3. unicorn_pid       # The file record current unicorn pid. used for server start&stop

Careful with file permissions. Nginx must be able to access `unicorn_sock_file`. There's also a very usual problem `Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock'`. This may caused by permission problem too.


## The end

Well. this is all. After I finished the work. The whole deployment looks very simple. All I have to do is make sure `unicorn -e production` run correctly on server just like how you build local development environment. Then connect Nginx and Unicorn. Finally link a domain name to your server IP.

Before this practice, web server runs like a magic to me. _I don't know how, but it works_. But there's no magic in coding world :p.
