---
layout: post
title: "HashiCorp Binary File Installer"
subtitle: "A quick fix for multiple machines"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge
I work on many different machines, and I hate re-installing the same software over and over again. I doubly hate it for binary files that don't install so much as get moved to a particular location in the file server. Since we have started using terraform for our IAC solution, I have had to download the binary onto four different machines, add the file location to the path, and move the executable to that location. In addition, terraform updated from 0.9.5 to 0.9.6, meaning that every machine needed a new binary downloaded!

# The Solution
The full code can be found [here](https://gist.github.com/tjsullivan1/8e489e68f3b7b5c0827540795cef2608). 

This was a quick and dirty tool to go to HashiCorp's web site to download all of the binaries and move them to a specific location. It will require sudo privileges to run correctly on Linux, and you will need administrator privileges on Windows to add "C:\Tools" to your path.

Thankfully, HashiCorp is very standard in its practices and all of its products follow a standard format: 
`url = 'https://releases.hashicorp.com/' + product + '/' + version + '/' + product + '_' + version + '_' + os_ver + '_amd64.zip'`. There are four main sections of this script. First, I configure logging as demonstrated by Al Sweigart. Then, I detect whether or not the machine is running Windows (if not, I assume Linux). If you are running Windows, the binary needs to have '.exe' appended. I then also ensure that C:\Tools or /usr/local/bin exist.

Next, I create a dictionary of all of the tools that we are going to download the binaries for as well as their version. This will allow me to quickly update version numbers upon release, or add and remove new HashiCorp tools. 

Finally, we loop through our dictionary and build the URL for each product. We then open the url and write the binary using shutil to copy the request response to the file. Now, we have a zip file, so I use zipfile to extract the file (e.g., packer or packer.exe depending on OS). Then, we set the Linux file permissions to add the executable bit for user, group, and other (the equivalent of chmod +x), in order to allow us to execute them without a full path. 

# Up and Running
You will need Python 3 to run this script. 

## Windows
On Windows, you will need to make C:\Test part of your path. There are numerous guides on how to modify the path manually on the internet. The quickest way I have found is to launch PowerShell as an admin and run: ` `. If you have a good, clean way to do this with Python, [let me know](https://tjsullivan1.github.io/contact)!

Then, to invoke the script, you can run `python install_hashi_tools.py`. Note, this script will create a log file in the invocation directory. 

## Linux 
On Linux, I would assume /usr/local/bin is in your path. The script invocation is a little less straightforward, make sure you run `sudo python3 install_hashi_tools.py`. Sudo is important to copy the files to /usr/local/bin, and the 3 is critical on RHEL machines that have Python 2 installed by default. 
