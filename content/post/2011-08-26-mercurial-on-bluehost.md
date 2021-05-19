---
author: Saman Barghi
categories:
- Uncategorized
comments: true
date: '2011-08-26'
permalink: /2011/08/26/mercurial-on-bluehost/
title: Mercurial on Bluehost
url: /2011/08/26/mercurial-on-bluehost
---

<p style="text-align: left;">
  I had to install a mercurial repository on bluehost for one of my projects. The good thing about mercurial is that it allows you to access your repository through http, and it is an ideal version control system for shared hosts. I googled around for it, and I found a couple of instructions available (<a href="http://bugtracker.gttools.com/public/wiki/bluehost/Mercurial">this</a> and <a href="http://gttools.com/bluehost-setup/mercurial-and-trac-setup-on-bluehost">this</a>). But they are outdated, and probably worked with earlier versions of mercurial. So I decided to rewrite the instruction of installing mercurial on your shared host (specifically bluehost):
</p>

<p style="text-align: left;">
  &nbsp;
</p>

<p style="text-align: left;">
  <span style="text-align: -webkit-auto;">Update your bash profile, add/modify these 3 lines:</span>
</p>

 <!--more-->
```
vim ~/.bash_profile
export LD_LIBRARY_PATH=”$HOME/packages/lib:$LD_LIBRARY_PATH”
export PYTHONPATH=”$HOME/packages/lib/python2.3/site-packages:$PYTHONPATH”
export PATH=”$HOME/packages/bin:$HOME/bin:$PATH”
source ~/.bash_profile
vim ~/.bashrc
PATH=$PATH:$HOME/bin:$HOME/packages/mercurial
source ~/.bashrc
```

Create folders (Please note, from now on my assumption is that your web directory is located at $HOME/public_html, please change the related parts if your configuration is different):

```
cd
mkdir install_files
mkdir packages
mkdir ~/public_html/hg
mkdir ~/public_html/hg/repos
```

Get Mercurial and install:

```
cd ~/packages
wget http://selenic.com/repo/hg-stable/archive/tip.tar.gz
tar zxf tip.tar.gz
mv tip.tar.gz ~/install_files/
mv Mercurial* mercurial
cd mercurial
#dont' forget to change the username
echo -e "[ui]\nusername=YOURUSERNAME &lt;YOUREMAIL@WEB.com&gt;" &gt; ~/.hgrc﻿
make local
./hg debuginstall
```

Running the last command should not show any problems and you would see :<span style="font-family: Consolas, Monaco, 'Courier New', Courier, monospace; font-size: 12px; line-height: 18px; white-space: pre;"> </span>

```
Checking encoding (UTF-8)...
Checking installed modules (~/packages/mercurial/mercurial)...
Checking templates...
Checking commit editor...
Checking username...
No problems detected
```

In order to be able to push, mercurial provides a cgi script(hgweb.cgi) to handle push commands (<http://mercurial.selenic.com/wiki/PublishingRepositories#hgweb>). In addition, there is a need for apache authentication module to perform the authentication tasks and manage accesses to our repository. Before, we can use the repository we need to copy the hgweb.cgi script to our web directory, and customize it afterwards:

```
sed 's|#import sys|import sys|g;s|/path/to/python/lib|'$HOME'/packages/mercurial|g;s|/path/to/repo/or/config|'$HOME'/public_html/hg/hgweb.config|g' ~/packages/mercurial/hgweb.cgi &gt; $HOME/public_html/hg/hgweb.cgi
chmod 755 ~/public_html/hg/hgweb.cgi
```

Create hgweb.config file. You can use *[collections] *instead of *[paths]* here, it depends on your personal taste. [Here][1] suggested to use paths instead of collection. So if you prefer to use *[collections]* instead, do not forget to remove all the lines related to hgweb.config when you initialize your repository:

```
echo [web] &gt; ~/public_html/hg/hgweb.config
echo allowpull=true &gt;&gt; ~/public_html/hg/hgweb.config
echo [paths] &gt;&gt; ~/public_html/hg/hgweb.config
```

Create .htaccess file:

```
echo 'Options +ExecCGI' &gt; ~/public_html/hg/.htaccess
echo 'RewriteEngine On' &gt;&gt; ~/public_html/hg/.htaccess
echo 'RewriteBase /hg' &gt;&gt; ~/public_html/hg/.htaccess
echo 'RewriteRule ^$ hgweb.cgi [L]' &gt;&gt; ~/public_html/hg/.htaccess
echo 'RewriteCond %{REQUEST_FILENAME} !-f' &gt;&gt; ~/public_html/hg/.htaccess
echo 'RewriteCond %{REQUEST_FILENAME} !-d' &gt;&gt; ~/public_html/hg/.htaccess
echo 'RewriteRule (.*) hgweb.cgi/$1 [QSA,L]' &gt;&gt; ~/public_html/hg/.htaccess
echo 'AuthUserFile /home/'$USER'/etc/hg-basic-auth' &gt;&gt; ~/public_html/hg/.htaccess
echo 'AuthName "HG Repositories"' &gt;&gt; ~/public_html/hg/.htaccess
echo 'AuthType Basic' &gt;&gt; ~/public_html/hg/.htaccess
echo 'Require valid-user' &gt;&gt; ~/public_html/hg/.htaccess
```

Create Passwd files:

```
cd
htpasswd -b -c -d ~/etc/hg-basic-auth HgUserName PASSWORD
```

For extra users:

```
htpasswd -b -d ~/etc/hg-basic-auth HgUserNameExtra PASSWORD
```

Initialize repositories (Do not forget to change all instances of PROJECT to your own project name):

```
cd ~/public_html/hg/repos
~/packages/mercurial/hg init PROJECT
echo 'PROJECT = '$HOME'/public_html/hg/repos/PROJECT' &gt;&gt; ~/public_html/hg/hgweb.config
```

For extra projects :

```
~/packages/mercurial/hg init ExtraPROJECT
echo 'ExtraPROJECT  = '$HOME'/public_html/hg/repos/ExtraPROJECT' &gt;&gt; ~/public_html/hg/hgweb.config
```

Now lets create hgrc files for each project :

```
echo '[web]' &gt; ~/public_html/hg/repos/PROJECT/.hg/hgrc
echo 'contact=admin email address' &gt;&gt; ~/public_html/hg/repos/PROJECT/.hg/hgrc
echo 'description=My releases' &gt;&gt; ~/public_html/hg/repos/PROJECT/.hg/hgrc
echo 'allow_push=USER1, USERn' &gt;&gt; ~/public_html/hg/repos/PROJECT/.hg/hgrc
echo 'allow_archive=zip' &gt;&gt; ~/public_html/hg/repos/PROJECT/.hg/hgrc
```

> By default, pushing is only allowed via HTTPS. To permit HTTP pushing you have to add this to your repository&#8217;s <tt>.hg/hgrc</tt> file (or your Web server user&#8217;s <tt>.hgrc</tt> file, such as<tt>/home/www-data/.hgrc</tt>, or a system-wide <tt>hgrc</tt> file like <tt>/etc/mercurial/hgrc</tt>):

I personally decided to put this in the hgrc file in my repository:

```
echo 'push_ssl = false' &gt;&gt; ~/public_html/hg/repos/PROJECT/.hgrc
```

You are basically done. You can access your repository using :

```
hg clone http://yourdomain.com/hg/PROJECT

```

<span class='st\_facebook' st\_title='Mercurial on Bluehost' st_url='http://www.samanbarghi.com/2011/08/26/mercurial-on-bluehost/'></span><span st\_via='saman\_b' class='st\_twitter' st\_title='Mercurial on Bluehost' st_url='http://www.samanbarghi.com/2011/08/26/mercurial-on-bluehost/'></span><span class='st\_email' st\_title='Mercurial on Bluehost' st_url='http://www.samanbarghi.com/2011/08/26/mercurial-on-bluehost/'></span><span class='st\_sharethis' st\_title='Mercurial on Bluehost' st_url='http://www.samanbarghi.com/2011/08/26/mercurial-on-bluehost/'></span><span class='st\_fblike' st\_title='Mercurial on Bluehost' st_url='http://www.samanbarghi.com/2011/08/26/mercurial-on-bluehost/'></span><span class='st\_plusone' st\_title='Mercurial on Bluehost' st_url='http://www.samanbarghi.com/2011/08/26/mercurial-on-bluehost/'></span><span class='st\_pinterest' st\_title='Mercurial on Bluehost' st_url='http://www.samanbarghi.com/2011/08/26/mercurial-on-bluehost/'></span>

 [1]: http://mercurial.selenic.com/wiki/PublishingRepositories#Configuration_of_hgweb