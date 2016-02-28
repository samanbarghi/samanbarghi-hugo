---
author: Saman Barghi
categories:
- FreeBsd
comments: true
date: '2011-02-04'
permalink: /2011/02/04/ssh-to-virtualbox-3-freebsd-guests/
title: SSH to VirtualBox 3 FreeBsd Guests
url: /2011/02/04/ssh-to-virtualbox-3-freebsd-guests
---

This post is based on <a href="http://muffinresearch.co.uk/archives/2010/02/08/howto-ssh-into-virtualbox-3-linux-guests/" target="_blank">&#8220;Howto: SSH into VirtualBox 3 Linux Guests&#8221;</a> by <a href="http://muffinresearch.co.uk/" target="_blank">Stuart Colville</a>, which I have updated to work for FreeBSD Guests: In older versions of VirtualBox one had to use a bridge interface to make this work. However, in newer versions; a virtual interface can be added to the Host by default, which make it way easier to access the guest machine through the host.

Usually, you have a primary network interface which uses NAT adapter, that one is needed for connecting to the Internet. What we are going to do is to add an additional interface to the guest machine. All you have to do is to access the settings of the guest when it is off.  Select Network->Adapter 2, then check the &#8220;Enable Network Adapter&#8221; as shown below. Change &#8220;attached to&#8221; to &#8220;Host-only Adapter&#8221; this will have the name vboxnet0 by default.

[<img class="size-full wp-image-143 alignnone" title="VirtualBox-FreeBsd" src="http://www.samanbarghi.com/wp-content/uploads/2011/02/VirtualBox-FreeBsd.png" alt="" width="501" height="397" />][1]

Now if you do *ifconfing* on your host machine (On Windows hosts use *ipconfig*), there will be a new interface called *vboxnet0 *(same as the virtual adapter name you just added). You will notice the IP address is something similar to this:

<pre class="brush: bash; title: ; notranslate" title="">inet addr:192.168.56.1  Bcast:192.168.56.255  Mask:255.255.255.0</pre>

Which means his interface can access all the IP addresses in the following range: *192.168.56.1-192.168.56.254*. This is needed to setup the guest interface properly.

Now, boot the FreeBsd guest. If you do *ifconfig *on your guest machine*, *You will notice that a second interface has been added to your machine (The name is *em1 *if you already had one interface). You have to configure the new interface; so you can access it by your host machine. We will set a static IP address for this interface using the IP range above.

Next add the following lines to *&#8220;/etc/rc.conf&#8221;* in your guest machine:

<pre class="brush: bash; title: ; notranslate" title="">ifconfig_em1="inet 192.168.56.10 netmask 255.255.255.0"
</pre>

Save this and run the following command to restart the network interfaces:

<pre class="brush: bash; light: true; title: ; notranslate" title="">/etc/rc.d/netif restart </pre>

If you have not enabled SSH on your FreeBsd guest yet, edit */etc/rc.conf* and add the following:

<pre class="brush: bash; title: ; notranslate" title="">sshd_enable="YES" </pre>

and then run:

<pre class="brush: bash; light: true; title: ; notranslate" title="">/etc/rc.d/sshd start </pre>

That&#8217;s it, now you should be able to ssh to your guest machine by:

<pre class="brush: bash; light: true; title: ; notranslate" title="">ssh saman@192.168.56.10 </pre>

.

<span class='st\_facebook' st\_title='SSH to VirtualBox 3 FreeBsd Guests' st_url='http://www.samanbarghi.com/2011/02/04/ssh-to-virtualbox-3-freebsd-guests/'></span><span st\_via='saman\_b' class='st\_twitter' st\_title='SSH to VirtualBox 3 FreeBsd Guests' st_url='http://www.samanbarghi.com/2011/02/04/ssh-to-virtualbox-3-freebsd-guests/'></span><span class='st\_email' st\_title='SSH to VirtualBox 3 FreeBsd Guests' st_url='http://www.samanbarghi.com/2011/02/04/ssh-to-virtualbox-3-freebsd-guests/'></span><span class='st\_sharethis' st\_title='SSH to VirtualBox 3 FreeBsd Guests' st_url='http://www.samanbarghi.com/2011/02/04/ssh-to-virtualbox-3-freebsd-guests/'></span><span class='st\_fblike' st\_title='SSH to VirtualBox 3 FreeBsd Guests' st_url='http://www.samanbarghi.com/2011/02/04/ssh-to-virtualbox-3-freebsd-guests/'></span><span class='st\_plusone' st\_title='SSH to VirtualBox 3 FreeBsd Guests' st_url='http://www.samanbarghi.com/2011/02/04/ssh-to-virtualbox-3-freebsd-guests/'></span><span class='st\_pinterest' st\_title='SSH to VirtualBox 3 FreeBsd Guests' st_url='http://www.samanbarghi.com/2011/02/04/ssh-to-virtualbox-3-freebsd-guests/'></span>

 [1]: http://www.samanbarghi.com/wp-content/uploads/2011/02/VirtualBox-FreeBsd.png