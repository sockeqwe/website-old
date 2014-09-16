---
layout: post
published: true
title: Hosting websites on github
mathjax: false
featured: false
comments: true
headline: Hosting websites with jekyll on github
categories:
  - personal
tags: personal
imagefeature: cover_github_family.jpg
---

Finally I have found time to setup my own website with blog. To be honest, I had to buy a domain for maven central to be able to upload my libraries with the groupid _"com.hannesdorfmann"_. Since I own now this domain I guess it's a good idea to provide some content.

There are some cool blog hosting services out there like [medium](http://www.medium.com), [tumblr](http://www.tumblr.com) or traditionally services like [blogger](www.blogger.com) or [blogspot](www.blogspot.com). Even wordpress is quite good and simple to setup. However I was looking for an easy way to publish content. Especially for technical articles (I have plans to write some if I find some time),  good and easy code highlightning is required. I'm a big fan of github pages. That's the reason why I'm already familar with markdown and jekyll. Hence I have chosen to publish my website with [jekyll](http://jekyllrb.com). Basically, jekyll transforms markdown (or plaintext) to static html websites. Additionaly you can **host jekyll based websites on github for free** and all your website (and blog) is under revision control. Furthermore readers who have found an error can simply fork this blog, solve the error and make a pull request. So overall, jekyll + github seems to me to be a good choice.

In this next few steps I will describe how easy it is to setup jekyll and to setup github to work with your own domain (in my case with www.hannesdorfmann.com):

1. Download and install jekyll on your local machine. It's easy: type `gem install jekyll` in your console (or visit [www.jekyllrb.com](http://jekyllrb.com/))
2. If you don't want to use the default theme check out this site: [www.jekyllthemes.org](http://jekyllthemes.org)
3. Sign up and sign in on github.
4. Visit [https://pages.github.com](https://pages.github.com/) where you will find more details how to setup a repostitory for your website hosted by github. Basically you have to create a new github repository with the name `username.github.io`, where _username_ is your github username. Commiting to this repository (master branch) will internally trigger jekyll on githubs server to generate static html sites. Your website will be visible to everyone out there by calling `http://username.github.io` (_username_ needs to replaces with your real github username). **That's all.**
5. Next (optional) I want to "link" (redirect) my domain to the this website hosted by github. That's also pretty simple and the guys at github did a realy good job. There are two more steps to do:
   1. You have to add and commit a `CNAME` file (with all caps!) to the root folder in your master branch. The CNAME file contains a single line with your domain. For instance my CNAME looks as follows:
`hannesdorfmann.com`. You can also use subdomains in you CNAME file like `blog.hannesdorfmann.com`.
   2. As last step you have to configure your DNS provider: Add an `A record` or `CNAME record`. Note that not every DNS provider allows to setup `A records`.


- **A record** Mine (http://inwx.com/) allows to setup A records. So remove previous A records and point _@_ to the following ip addresses (Note that ip addresses may change in the future. Check [github](https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider) page to get the current ones):

> @: 192.30.252.153 <br/>
> @: 192.30.252.154

- **CNAME** is something like an alias to link to another url. You have to link your domain to username.github.io. **(Note the trailing full stop)**. The CNAME records looks like this:

> www: username.github.io.

Now you have to wait until the previous DNS entry expires (TTL = Time To Live) to open the github hosted website with your browser. In the meantime you can check your setup from console with the following command:

> dig yourdomain.com +nostats +nocomments +nocmd

**Thank you github for free hosting!**
