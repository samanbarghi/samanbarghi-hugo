---
author: Saman Barghi
categories:
- Geeky
- Scripts
comments: true
date: '2012-08-29'
permalink: /2012/08/29/national-geographic-photo-of-the-day-downloader-for-linux/
title: National Geographic Photo of the day Downloader for Linux
url: /2012/08/29/national-geographic-photo-of-the-day-downloader-for-linux
---

[<img class="wp-image-252 alignleft" title="National Geographic Photo Of the Day" src="http://www.samanbarghi.com/wp-content/uploads/2012/08/57275_1600x1200-wallpaper-cb1343743721-300x225.jpg" alt="National Geographic Photo Of the Day" width="300" height="225" />][1] I am a fan of National Geographic photos on their site, and I also get bored of by my desktop background after a while. So I decided to create a script to download National Geographic photo of the day, and using it as my desktop background. I am using it over Gnome3 (I am using Fedora as I think it&#8217;s more stable than Ubuntu, and I like Gnome3 way better than Unity), but if you are a Unity user it should work for you as well. You can find the script here:

> <a title="https://github.com/samanbarghi/ngphotodownloader" href="https://github.com/samanbarghi/ngphotodownloader" target="_blank">https://github.com/samanbarghi/ngphotodownloader</a>

Although National Geographic posts a photo everyday, not all the photos come with a high quality format. So the script checks whether a wallpaper format exists or not. If so, it downloads the photo into the same directory the script resides.

&nbsp;

## Setup

All you need to do is  create a directory for your wallpaper and put the script in there, e.g.:

```
cd ~/Pictures/
git clone https://github.com/samanbarghi/ngphotodownloader.git NGWallpapers
```

Simply run the script to get the Photo of the day. But doing that manually everyday is not fun. Here cron comes handy. You need to run the script at least once each day, to automate the process you can use cron to download the script and set it as your desktop background: 

```
0 12 * * * sh /home/yourusername/Pictures/NGWallpapers/ngwallpaper.sh
```

In my case since I am running the script on my laptop, and my laptop is not always on; I call the script every 3 hours to make sure it runs at least once each day. Don&#8217;t worry about duplicates, the script will not download the image if it already exists in the directory. The overhead of the script on cpu/memory/network is negligible, so don&#8217;t worry about calling the script 8 times a day:

```
00 */3 * * * sh /home/yourusername/Pictures/NGWallpapers/ngwallpaper.sh
```

Enjoy! <br />

[1]: http://www.samanbarghi.com/wp-content/uploads/2012/08/57275_1600x1200-wallpaper-cb1343743721.jpg