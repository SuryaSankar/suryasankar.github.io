---
layout: post
title:  "How to setup a blog as a subfolder of a domain using nginx"
date:   2017-11-29 00:26:37 +0530
categories: wordpress nginx
canonical_url: 'https://medium.com/the-inkmonk/setup-wordpress-blog-as-domain-subfolder-using-nginx-ad5346f082e4'
---

We had initially hosted our blog as a subdomain of inkmonk.com. It was easier to set it up that way. Just 2 steps.
Get a managed wordpress instance on AWS from Bitnami
Point the DNS entry on Cloudflare for the blog. subdomain to this bitnami instance's IP.

That's it.
But over time, our marketing team expressed reservations about this set up. They wanted to move the blog inside a sub-folder to gain SEO boost from the blog posts. I was initially about to argue that the sub-domain vs sub-folder debate was not important anymore, since Google claimed it didn't care about that. But after some digging around, I found this Moz Forum Post which drew the conclusion that Google's claims do not yet match the reality and that keeping the blog inside the same domain does provide a significant SEO boost.
So with that established, I had to figure out the best way to do this.
I wasn't going to install a wordpress instance in our production servers in order to serve the blog. Firstly, it would make the servers vulnerable to the widely known wordpress exploits. Secondly, it would reduce our deployment flexibility. Currently our production servers can be spun up and down as per requirement since the persistent data is kept separately. But with the introduction of wordpress, that could change. I was not sure how replicable a wordpress setup was, whether it could be added to our ansible scripts to deploy at will etc.
Then I realized that all I had to do was to use nginx as a reverse proxy. I could continue running wordpress on the same Bitnami instance and just add a line to proxy a sub-folder alone to that instance. After a facepalm for not realizing this sooner, I just added these 2 lines inside the server directive of the site's nginx conf

```
location /blogs/printing/ { 
    proxy_pass https://blog.inkmonk.com/; 
}
```

Now https://inkmonk.com/blogs/printing/ was showing the contents of blog.inkmonk.com as expected. But the job was not done yet. There were still a few problems

###Problem 1 - Links were still pointing to blog.inkmonk.com
 All the links in the content were still pointing to blog.inkmonk.com. The problem of course was that the wordpress blog was still configured to use that as the site url. The wp-config.php in the wordpress installation had to be edited to comment out the values of 2 entries and replace them with the new urls

```
/*
define('WP_SITEURL', 'http://' . $_SERVER['HTTP_HOST'] . '/'); define('WP_HOME', 'http://' . $_SERVER['HTTP_HOST'] . '/'); 
*/ 
define('WP_SITEURL', 'https://inkmonk.com/blogs/printing/'); define('WP_HOME', 'https://inkmonk.com/blogs/printing/');
```

This solved the links problem. Now all the links in the blog were using the new urls.

###Problem 2 - Mixed content problem due to SSL
Our main domain uses SSL. But the wordpress blog was not on SSL. This caused a problem with stylesheets and scripts not being downloaded. Took me a while to realize that this was the issue. Once found, the solution was simple. Just change the nginx proxy pass conf to use https. And the url was changed to use the IP address of the instance instead of blog.inkmonk.com as well

```
location /blogs/printing/ { 
    proxy_pass https://w.x.y.z/blogs/printing/; 
}
```

###Problem 3 - Wp-admin and Wp-login not working properly
 There was still one more problem. Whenever a form was submitted in the wp-admin dashboard, the url was changing to inkmonk.com/wp-admin.php instead of inkmonk.com/blogs/printing/wp-admin.php. Login form also had similar issues. This was a headache. Finally I realized that the problem was that the wordpress installation was mounted at the DocumentRoot of the Apache server in the bitnami instance. It had to be aliased to a subfolder /blogs/printing mirroring the url structure.
 This was done by changing the httpd-prefix.conf inside the conf folder in the bitnami wordpress app. The existing DocumentRoot line had to be commented out and 2 aliases had to be added

```
#DocumentRoot "/opt/bitnami/apps/wordpress/htdocs" 
Alias /blogs/printing/ "/opt/bitnami/apps/wordpress/htdocs/" 
Alias /blogs/printing "/opt/bitnami/apps/wordpress/htdocs"
```

After doing this, the nginx conf of the main server also had to be updated to use the sub-folder url instead of the document root.

```
location /blogs/printing/ { 
    proxy_pass https://w.x.y.z/blogs/printing/; 
}
``
`
And with this the job was finally done. The blog had been successfully hosted as a subdomain. There was still one job pending. The original blog.inkmonk.com urls had to be given a 301 redirect to the new urls. This should not be done at the DNS level - since that will redirect the domain name only. The correct option is to use nginx again.
A new conf for blog.inkmonk.com was added in the site-enabled list of the nginx conf like this

```
server {
 listen 80;
 server_name blog.inkmonk.com;
 return 301 https://inkmonk.com/blogs/printing$request_uri; 
}
```

After this, the DNS entry of blog.inkmonk.com was changed to point to the instance which hosted this nginx config instead of the IP of the bitnami instance. Now the traffic would hit this nginx entry and will be served a 301 permanent redirect to the new urls.
And that wrapped up the task.