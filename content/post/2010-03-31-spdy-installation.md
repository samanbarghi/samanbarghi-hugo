---
author: Saman Barghi
categories:
- Guides
comments: true
date: '2010-03-31'
permalink: /2010/03/31/spdy-installation/
title: SPDY installation
url: /2010/03/31/spdy-installation
---

I am taking a course called &#8220;Latency in Communication Systems&#8221; taught by <a href="http://www.cs.uwaterloo.ca/~mkarsten/" target="_blank">Martin Karsten</a> in University of Waterloo. We talked about different aspects of latency in communication systems, there were some lectures from industry (Sandvine, TD Securites, RIM, and SUN/Oracle) about what were the different problems they were facing during their system implementation. I had also presented two papers on profiling and router performance during the course. My course project is the reason I am writing this post, I have to install and evaluate <a title="SPDY" href="http://www.chromium.org/spdy" target="_blank"><strong>SPDY</strong></a>. SPDY is a new application-layer protocol designed by Google to replace HTTP in order to make the web faster. It offers multiple requests per connection, server-initiated requests along with the client-initiated requests, header and data compression over SSL. In this post I am not going to talk about SPDY and it&#8217;s features, I will explain the process of SPDY client (chromium) and server installation as I need them later to evaluate this protocol.

## Install Chromium

Edit: Follow the build instruction for linux here: <http://code.google.com/p/chromium/wiki/LinuxBuildInstructions>

I started to compile chromium on Ubuntu Karmic Koala 9.10. The obvious place to start is the <a title="Chromium" href="http://dev.chromium.org/" target="_blank">chromium website</a>, Although take a look at <a title="Cache data for SPDY server" href="http://groups.google.com/group/spdy-dev/browse_thread/thread/16e0a9d5592ca908" target="_blank">this</a>, we need to enable data recording to cache some data for the in-memory server, I will explain this later. These are the required step to take :

1.  Install all the <a title="Chromium Ubuntu prerequisites" href="http://code.google.com/p/chromium/wiki/LinuxBuildInstructionsPrerequisites#Ubuntu_Setup" target="_blank">prerequisites</a>
2.  Download the source tarball from <a title="Chromium Source tarball" href="http://build.chromium.org/buildbot/archives/chromium_tarball.html" target="_blank">here</a>.
3.  Extract it to a location (It should not include any spaces). I extracted it to <tt>~/chromium</tt>
4.  Install depot_tools 
    1.  <pre class="brush: bash; light: true; title: ; notranslate" title="">svn co http://src.chromium.org/svn/trunk/tools/depot_tools </pre>
    
    2.  Add depot_tools to your PATH: <pre class="brush: bash; light: true; title: ; notranslate" title="">$ export PATH=`pwd`/depot_tools:"$PATH" </pre>

5.  Updating your checkout once by running <pre class="brush: bash; light: true; title: ; notranslate" title="">gclient sync --force </pre>
    
    <span style="font-family: arial,sans-serif;"> in the source code directory (~/chromium/src). It is important to include the &#8216;&#8211;force&#8217; option as I was getting this message: </span>
    
    <pre class="brush: plain; light: true; title: ; notranslate" title="">make: *** No rule to make target `third_party/yasm/source/patched-yasm/ modules/arch/x86/gen_x86_insn.py', needed by `out/Debug/obj/gen/ third_party/yasm/x86insns.c'. Stop." </pre>
    
    <span style="font-family: arial,sans-serif;">over and over again since I forgot to include the <tt>--force</tt> option. It seems that this command generate platform-specific files</span></li> 
    *   **Important:** ensure that you have <tt>GYP_GENERATORS=make</tt> in your environment before running <tt>gclient sync</tt> or <tt>gclient runhooks --force</tt>. This tells the Chromium build system to output Makefiles. Example: <pre class="brush: bash; light: true; title: ; notranslate" title="">export GYP_GENERATORS=make
gclient sync</pre>
    
    *   Take this step only if you want to record data to be used by flip-server, if you just want to compile and run chromium skip this step and go to step 9. Modify the file <tt>chrome/common/chrome_constants.cc </tt>- change the line which reads <pre class="brush: cpp; light: true; title: ; notranslate" title="">kRecordModeEnabled = false</pre>
        
        to
        
        <pre class="brush: cpp; light: true; title: ; notranslate" title="">kRecordModeEnabled = true</pre>
    
    *   Go to the source directory and write (if you are using a multiple-core machine use <tt>-jX</tt>, where <tt>X</tt> is the number of make processes to startup): <pre class="brush: bash; light: true; title: ; notranslate" title="">make chrome </pre>
    
    *   You can find the executable in <tt>src/out/Debug/chrome</tt></ol> 
    Now you should be able to run chrome with no problem.
    
    ## Install flip-server
    
    To install flip-server, take a look at <a title="Flip server installation" href="http://www.chromium.org/spdy/running_flipinmemserver" target="_blank">this</a>. Go to the Chromuim <tt>src</tt> directory, and here are the steps that I took to install flip server:
    
    1.  If you don&#8217;t already have a test key.pem and certificate.pem, you can generate one like so: <pre class="brush: bash; light: true; title: ; notranslate" title="">openssl genrsa -out key.pem 1024
openssl req -new -key key.pem -out request.pem  #and answer the questions at the prompt with whatever
openssl x509 -req -days 30 -in request.pem -signkey key.pem -out cert.pem</pre>
    
    2.  Get the patch from: <http://codereview.chromium.org/1566013>, Click the &#8216;download raw patch&#8217; link and save. Assuming you&#8217;re in the <tt>chromium/src</tt> directory, type: <pre class="brush: bash; light: true; title: ; notranslate" title="">patch -p0 &lt; filename_of_the_patch_you_just_downloaded</pre>
    
    3.  Find the commented-out-section in net.gyp (look for &#8216;flip\_in\_mem\_edsm\_server&#8221;) and uncomment it.
    4.  You will need to install the openssl libraries. The exact packages depend on your linux distribution (but will be named something like ssl-dev or openssl-dev, or in ubuntu libssl-dev), so watch the error messages, if any. After you&#8217;ve done all that, you do: <pre class="brush: bash; light: true; title: ; notranslate" title="">make out/Debug/flip_in_mem_edsm_server </pre>
    
    5.  I had to change these lines to get rid of some compiling errors : 
        *   While compiling I got :  
            > <pre class="brush: plain; light: true; title: ; notranslate" title="">/usr/include/linux/tcp.h:72: error: '__u32 __fswab32(__u32)' cannot
appear in a constant-expression
/usr/include/linux/tcp.h:72: error: a function call cannot appear in a
constant-expression
/usr/include/linux/tcp.h:73: error: '__u32 __fswab32(__u32)' cannot
appear in a constant-expression
/usr/include/linux/tcp.h:73: error: a function call cannot appear in a
constant-expression
</pre>
            
            To fix it, in file <tt>src/net/tools/flip_server/flip_in_mem_edsm_server.cc</tt> change the line that reads:
            
            <pre class="brush: plain; light: true; title: ; notranslate" title="">#include &lt;linux/tcp.h&gt;</pre>
            
            to
            
            <pre class="brush: plain; light: true; title: ; notranslate" title="">#include &lt;netinet/tcp.h&gt;</pre>
        
        *   In the same file change : <pre class="brush: plain; light: true; title: ; notranslate" title="">state-&gt;ssl_method = SSLv23_method();</pre>
            
            to
            
            <pre class="brush: plain; light: true; title: ; notranslate" title="">state-&gt;ssl_method = const_cast&lt;SSL_METHOD *&gt; (SSLv23_method()); </pre>
        
        *   I was getting some errors related to <tt>printSslError </tt> function in <tt>flip_in_mem_edsm_server.cc</tt>, it seems that this function prints out the errors related to SSL encryption, since I did not need that I commented the contents of the function out.
    
    ## Install Chrome which is able to talk to localhost server
    
    Now if you want a quick chrome installation that is able to talk to your installed server on localhost you need to copy the following script, save it in a file and run <tt>bash filename</tt>:
    
    <pre class="brush: bash; collapse: true; light: false; title: ; toolbar: true; notranslate" title="">#!/bin/bash

if [ "$(dpkg --print-architecture)" == "amd64" ]; then
  echo -ne "\n\nYou have a 64 bit computer\n\n"
  linux_versions="linux64";
else
  echo -ne "\n\nYou have a 32 bit computer\n\n"
  linux_versions="linux";
fi

for linux_version in $linux_versions; do
  install_dir=$HOME/spdy-chrome-canary/$linux_version

  if mkdir -p $install_dir; then
    echo "Directory: \"$install_dir\" made or already existed"
  else
      echo "$install_dir exists, but is not a directory."
      echo "Please remove whatever is there so this script can proceed next time."
      exit 128
  fi

  pushd $install_dir

  install -d chrome-linux-old
  mv chrome-linux chrome-linux-old/chrome-linux-`date +"%F-%k-%M-%S-%N"`
  rm -rf chrome-linux chrome-linux-old chrome-linux.zip
  wget http://build.chromium.org/buildbot/continuous/$linux_version/LATEST/chrome-linux.zip
  unzip chrome-linux.zip
  rm -rf chrome-linux.zip

  popd

  filename_base=SPDY-Chrome-$linux_version
  cat &gt;&gt; $HOME/Desktop/$filename_base.desktop &lt;&lt;-EOF
[Desktop Entry]
Version=1.0
Encoding=UTF-8
Name=$filename_base
Categories=Application;Network;WebBrowser;
Exec=$install_dir/chrome-linux/chrome --use-spdy --enable-logging --log-level=0 --user-data-dir=.$filename_base %U
Icon=/tmp/chrome-linux/product_logo_48.png
MimeType=text/html;text/xml;
Terminal=false
Type=Application
EOF

  filename_base=SPDY-Chrome-local-server-$linux_version
  cat &gt;&gt; $HOME/Desktop/$filename_base.desktop &lt;&lt;-EOF
[Desktop Entry]
Version=1.0
Encoding=UTF-8
Name=$filename_base
Categories=Application;Network;WebBrowser;
Exec=$install_dir/chrome-linux/chrome --use-spdy --enable-logging --log-level=0 --user-data-dir=.$filename_base --host-resolver-rules='MAP * localhost' --testing-fixed-http-port=10040 --testing-fixed-https-port=10040 %U
Icon=/tmp/chrome-linux/product_logo_48.png
MimeType=text/html;text/xml;
Terminal=false
Type=Application
EOF

  filename_base=NO-SPDY-Chrome-local-server-$linux_version
  cat &gt;&gt; $HOME/Desktop/$filename_base.desktop &lt;&lt;-EOF
[Desktop Entry]
Version=1.0
Encoding=UTF-8
Name=$filename_base
Categories=Application;Network;WebBrowser;
Exec=$install_dir/chrome-linux/chrome --enable-logging --log-level=0 --user-data-dir=.$filename_base --host-resolver-rules='MAP * localhost' --testing-fixed-http-port=16002 --testing-fixed-https-port=16002 %U
Icon=/tmp/chrome-linux/product_logo_48.png
MimeType=text/html;text/xml;
Terminal=false
Type=Application
EOF

done
</pre>
    
    <span class='st\_facebook' st\_title='SPDY installation' st_url='http://www.samanbarghi.com/2010/03/31/spdy-installation/'></span><span st\_via='saman\_b' class='st\_twitter' st\_title='SPDY installation' st_url='http://www.samanbarghi.com/2010/03/31/spdy-installation/'></span><span class='st\_email' st\_title='SPDY installation' st_url='http://www.samanbarghi.com/2010/03/31/spdy-installation/'></span><span class='st\_sharethis' st\_title='SPDY installation' st_url='http://www.samanbarghi.com/2010/03/31/spdy-installation/'></span><span class='st\_fblike' st\_title='SPDY installation' st_url='http://www.samanbarghi.com/2010/03/31/spdy-installation/'></span><span class='st\_plusone' st\_title='SPDY installation' st_url='http://www.samanbarghi.com/2010/03/31/spdy-installation/'></span><span class='st\_pinterest' st\_title='SPDY installation' st_url='http://www.samanbarghi.com/2010/03/31/spdy-installation/'></span>