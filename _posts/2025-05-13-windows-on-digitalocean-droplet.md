---
layout: distill
title: How to run Windows 10/11 Professional on Digital Ocean Cloud Virtual Machine from Scratch
description: Steps to run a Windows machine on Ubuntu based Digital Ocean Droplet
tags: cloud-vm windows
giscus_comments: true
date: 2025-05-13
featured: true

authors:
  - name: Roshan

# bibliography: 2018-12-22-distill.bib

# Optionally, you can add a table of contents to your post.
# NOTES:
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - we may want to automate TOC generation in the future using
#     jekyll-toc plugin (https://github.com/toshimaru/jekyll-toc).

toc:
  - name: "Part 1: Create Ubuntu droplet (VM1) for preparing the Windows Image"
    url: "#part-1"
  - name: "Part 2: Prepare Windows Image"
    url: "#part-2"
  - name: "Part 3: Windows Configuration"
    url: "#part-3"
  - name: "Part 4: Compress Windows Hard Disk and Use it"
    url: "#part-4"
  - name: "Part 5: Create New Ubuntu droplet (VM2) for running the Windows"
    url: "#part-5"
  - name: "Part 6: Network Configuration in Windows"
    url: "#part-6"
  - name: "Part 7: Hurray, finally you have configured your Windows machine on droplet"
    url: "#part-7"
  - name: "Part 8: Allocate Unused Storage (Optional)"
    url: "#part-8"






# Below is an example of injecting additional post-specific styles.
# If you use this post as a template, delete this _styles block.
_styles: >
  .fake-img {
    background: #bbb;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  }
  .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }
  .zoom-img {
    width: 100%;
    max-width: 100%;
    transition: transform 0.3s ease;
    cursor: zoom-in;
  }

  .zoom-img.zoomed {
    transform: scale(1.4); /* adjust zoom level */
    cursor: zoom-out;
    z-index: 10;
  }
---

<a name="part-1"></a>

## Part 1: Create Ubuntu droplet (VM1) for preparing the Windows Image

### 1. Create Ubuntu Virtual Machine on DigitalOcean
- Choose Ubuntu 22, 24 or any LTS support version.
- Specs: **4 GB Memory** and **2 CPUs** as shown in the SS. This is minimum requirements I have taken for preparing Windows 10 Professional Image. You can take higher droplet specification if you want for Windows 11 Professional or any.
<!-- <a href="/assets/img/create_vm1.png" target="_blank">
  <img src="/assets/img/create_vm1.png" alt="VM1 Creation" style="width:100%; height:auto;">
</a> -->

<img src="/assets/img/blog/windows-on-digitalocean-droplet/create_vm1.png" alt="VM1 Creation" class="zoom-img" onclick="this.classList.toggle('zoomed')">

- Use **password-based login** instead of ssh login
- Set and remember the password for later login to this droplet. Lets call this droplet as VM1.

### 2. SSH into the VM
Open the terminal and SSH into the droplet (VM1) using the below command. Replace the \<VM_IP\> with the public IP assigned to your droplet.
```bash
$ ssh root@<VM1_IP>
```
- Enter the password which you set while creating the droplet (VM1).

### 3. Install Required Packages
After login to your droplet in Step 2, use the below commands and install the required packages 
```bash
$ apt-get update && apt-get upgrade -y
$ apt-get install xfce4 firefox xrdp gzip -y
$ apt-get install qemu qemu-utils qemu-system-x86 -y
$ echo xfce4-session > ~/.xsession
$ reboot
```

### 4. Connect to Droplet using local GUI client
-- **Linux**: If you are using Linux in your host machine and you have Remmina application, then follow the below steps for Linux. Otherwise first install Remmina using below commands (Below commands are for Ubuntu/Debian, you can find similar commands for your Linux distro):
```bash
$ sudo apt update
$ sudo apt install remmina
$ sudo apt install remmina remmina-plugin-rdp remmina-plugin-vnc remmina-plugin-secret
```

and then proceed with further steps for Linux.
- Open Remmina → Enter VM1 IP → Full screen as shown in SS → Username: `root`, Password: VM1 password

<img src="/assets/img/blog/windows-on-digitalocean-droplet/remmina_login_gui_ubuntu.png" alt="VM1 Creation" class="zoom-img" onclick="this.classList.toggle('zoomed')">

-- **Windows**: If you are using Windows OS in your host machine, then simply follow below steps:

Open Remote Desktop Connection → Enter VM1 IP → Username: `root` → Click Connect.

In xrdp session, enter the Username: `root` and Password: VM1 password


### 5. Download the following:
After login into the droplet, open the browser and download the following iso in the machine (it will be downloaded in `/root/Downloads`). I am preferring Windows 10 Pro for my work. You can download any Windows of you requirement. Also make sure your droplet should have that minimal requirements  specifications (RAM and CPU) for your preferred Windows:

- [VirtIO Drivers](https://github.com/virtio-win/kvm-guest-drivers-windows/wiki/Driver-installation)
- [Windows 10 ISO](https://www.microsoft.com/en-us/software-download/windows10ISO)


In case the download failing or any issue, close the RDP session and simply download these in your local PC and follow Step 6.


<!-- ## 🧩 Part 2: Download & Transfer ISO Files -->

### 6. Transfer ISO Files to the Droplet
If you are unable to download the files directly in the droplet in Step 5, you can download those iso files in your local PC and copy those into the droplet using below commands. The same commands will work Windows users too. Replace the \<VM_IP\> with your droplet public IP:

```bash
$ scp virtio-win-0.1.271.iso root@<VM1_IP>:/root/Downloads
$ scp Windows_10.iso root@<VM1_IP>:/root/Downloads
```

PS: This will take little longer based on your Internet Speed.

---

<a name="part-2"></a>

## Part 2: Prepare Windows Image 

### 7. Create a Virtual Disk
Open the terminal and SSH to your droplet (VM1) as done in Step 2. Run the below commands and create a virtual disk of the required size. This disk size will be the System Directory (Local Disk C) for installing Windows. You can give the required size as per the requirements for your preferred Windows you want to install. In my case I am installing Windows 10 Pro, where `80 GB` is good enough to install it.  

```bash
$ cd Downloads
$ qemu-img create -f raw wins_harddisk.raw 80G
$ ls
```
Check whether that all the three files are in this Downloads directory:
- wins_harddisk.raw
- Windows ISO
- Virtio ISO


### 8. Run QEMU Emulator
To install Windows in this Virtual Hard Disk, run the Qemu Emulator using the below command. Replace the \<windows.iso\> and \<virtio.iso\> with the corresponding iso file names:

```bash
$ qemu-system-x86_64 \
  -m 3000M \
  -cpu host \
  -enable-kvm \
  -boot order=d \
  -usbdevice tablet \
  -drive file=wins_harddisk.raw,format=raw,if=virtio \
  -drive file=/root/Downloads/<windows.iso>,media=cdrom \
  -drive file=/root/Downloads/<virtio.iso>,media=cdrom \
  -vnc :1
```
Replace \<VM_IP\> with public IP of the droplet.
- **Linux User**: Open Remmina Desktop Client → Select VNC → Enter `<VM1_IP>:5901` → Hit Enter
- **Windows User**: Download the RealVNC Viewer Application using below link: 
  - [RealVNC Viewer](https://www.realvnc.com/en/connect/download/viewer/) 
  - Open RealVNC and Enter `<VM1_IP>:5901` → Hit Enter

### 9. Load VirtIO Drivers
Choose the Custom installation method and reach to the screen of disk creation. You will not see any disk over there. We have to first install few drivers and then we will get the empty wins harddisk `80 GB` which we created in previous step. 

Click `Load Driver` → Enter the VirtIO CD (shown in SS) → Select Balloon 

<img src="/assets/img/blog/windows-on-digitalocean-droplet/select_driver.png" alt="Select Driver" class="zoom-img" onclick="this.classList.toggle('zoomed')">

→ Select w10 (Select according to your version of windows you are installing) → Select amd64 → Click OK 

<img src="/assets/img/blog/windows-on-digitalocean-droplet/select_windows.png" alt="Select Driver" class="zoom-img" onclick="this.classList.toggle('zoomed')">

→ select first path → Click Next to install

<img src="/assets/img/blog/windows-on-digitalocean-droplet/select_driver2.png" alt="Select Driver" class="zoom-img" onclick="this.classList.toggle('zoomed')">


-- Repeat the same for remaining drivers listed below from VirtIO CD:
- `Balloon`
- `NetKVM`
- `vioRNG`
- `VioSCSI`
- `VioStor`

-- After installing all the above drivers, the virtual disk will be displayed as shown below.

<img src="/assets/img/blog/windows-on-digitalocean-droplet/virtual_disk.png" alt="Virtual Disk" class="zoom-img" onclick="this.classList.toggle('zoomed')">

→ Select the disk → Click Next to install

<img src="/assets/img/blog/windows-on-digitalocean-droplet/windows_install.png" alt="Windows Install" class="zoom-img" onclick="this.classList.toggle('zoomed')">

→ Complete the installation and give a **Username** for your Windows machine and select a **STRONG** password (Use combination of alphabets, numbers, special symbols). Remember both the **Username** and **Password**.

**PS**: Strong **Username** and **Password** for Windows Machine will help protect from active scans and attacks.


---

<a name="part-3"></a>

## Part 3: Windows Configuration

### 10. Inside Windows Desktop
After installing the Windows, configure the below settings:

1. **Enable Remote Desktop**
   Go to Start menu, and search **Remote Desktop Settings**, hit Enter. Toggle it on.

2. **Change RDP Port**
   Change the RDP default port to some random port:
   - Open **regedit** from Start menu search
   - Navigate to:
     ```
     HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp
     ```
   - Modify **PortNumber** field value from **3389** to **18021** (Decimal)

3. **Firewall Rule**
   - Open **Windows Defender Firewall with Advanced Security** from Start menu.
   - Select all the inbound rules and disable all inbound rules.
   
   <img src="/assets/img/blog/windows-on-digitalocean-droplet/disable_all_firewall_rules.png" alt="Windows Install" class="zoom-img" onclick="this.classList.toggle('zoomed')">

   - Click on New Rule to add a new Inbound rule.

   <img src="/assets/img/blog/windows-on-digitalocean-droplet/select_port.png" alt="Windows Install" class="zoom-img" onclick="this.classList.toggle('zoomed')">

   - Allow inbound TCP port **18021** (Domain, Private, Public)

   <img src="/assets/img/blog/windows-on-digitalocean-droplet/port_no.png" alt="Windows Install" class="zoom-img" onclick="this.classList.toggle('zoomed')">
   
   - Name: **Rule 18021 for RDP**


4. **Disable Ctrl+Alt+Del requirement at lock screen**
   - Open **Local Security Policy** in the installed Windows.
   - Navigate to Local Policies\Security Options.
   - Find the Interactive logon option for CTRL+ALT+DEL. (as shwon in SS below) 

   <img src="/assets/img/blog/windows-on-digitalocean-droplet/local_security_policy.png" alt="Windows Install" class="zoom-img" onclick="this.classList.toggle('zoomed')">
  
   - Double Click and Enable it.
   
   <img src="/assets/img/blog/windows-on-digitalocean-droplet/enable_ctrl_alt_del.png" alt="Windows Install" class="zoom-img" onclick="this.classList.toggle('zoomed')">
  

5. **Shutdown the machine**

---

<a name="part-4"></a>

## Part 4: Compress Windows Hard Disk and Use it
 Login to the droplet using SSH as done in Step 2. Go to the Downloads folder where you have your **wins_harddisk.raw** file.

### 11. Gzip the Virtual Hard Disk

Compress the **wins_harddisk.raw** file and give a name as shown in below command.

```bash
$ dd if=wins_harddisk.raw status=progress | gzip -c > windows2010.gz
```

<img src="/assets/img/blog/windows-on-digitalocean-droplet/gzip_harddisk.png" alt="compress disk" class="zoom-img" onclick="this.classList.toggle('zoomed')">
  

### 13. Install Apache and Move `.gz` the compressed file to Web Directory
We will use Apache to download this compressed disk file to the droplet where we want to install and run Windows.

```bash
$ apt install apache2
$ ufw allow 'Apache'
$ cp windows2010.gz /var/www/html/
```
You can preserve this file, store it in your any web location from you can easily download for later use.


---

<a name="part-5"></a>

## Part 5: Create New Ubuntu droplet (VM2) for running the Windows
This is crucial step where we are actually going to uncompress the `.gz` windows disk and start using Windows.

### 14. Create New Droplet (VM2) on DigitalOcean
Select the minimum requirements according to your Windows you chose in previous step. Since I used Windows 10 Pro. The following specs was good enough for me to run it. The same steps to be followed as in Step 1 for creating droplet.

- Specs: 4GB RAM, 2 CPUs
- Turn off the droplet.
- Go to **Recovery Tab** → Boot from **Recovery ISO**

<img src="/assets/img/blog/windows-on-digitalocean-droplet/switch_to_recovery_boot.png" alt="switch to recovery boot" class="zoom-img" onclick="this.classList.toggle('zoomed')">
  
- Turn droplet **on**
- Open **Recovery Console**



### 15. In Recovery Console
1. Press **6** to get bash prompt for Interactive Shell. 

   <img src="/assets/img/blog/windows-on-digitalocean-droplet/recovery_console.png" alt="recovery console" class="zoom-img" onclick="this.classList.toggle('zoomed')">

2. *Check disk name*:

    Run the below command and check the disk name whether its vda or sda. In my case, its vda.
```bash
$ ls -la
$ lsblk
```

3. *Download and write Windows disk*:

    Replace the \<vda/sda\> with vda or sda depends on you disk name.

```bash
$ wget -O- http://<VM1_IP>/windows2010.gz | gunzip | dd of=/dev/<vda/sda>
```

   <img src="/assets/img/blog/windows-on-digitalocean-droplet/download_windows.png" alt="download windows" class="zoom-img" onclick="this.classList.toggle('zoomed')">


### 16. Switch back to Boot from Hard Drive

  After the download finishes, you are done:

- **Turn off droplet (VM2)**
- Switch boot to **Hard Drive**
   <img src="/assets/img/blog/windows-on-digitalocean-droplet/switch_back_to_boot_from_hard_drive.png" alt="switch to hard drive" class="zoom-img" onclick="this.classList.toggle('zoomed')">

- Turn **on** machine

---

<a name="part-6"></a>

## Part 6: Network Configuration in Windows

To configure internet in the Windows machine on droplet VM2, follow the below steps:

1. Go to Access tab and click **Launch Recovery Console** to open Windows in recovery mode.
   
   <img src="/assets/img/blog/windows-on-digitalocean-droplet/launch_recovery_console.png" alt="launch recovery console" class="zoom-img" onclick="this.classList.toggle('zoomed')">

2. Go to **View Network Connections** from Start menu.
3. Double click first Ethernet interface **Ethernet Instance 0 2** and go to **Properties**.
4. Go to **Internet Protocol Version 4 (TCP/IPv4) > Properties** and give the IPs and Gateway manually, same as displayed at the bottom.

   <img src="/assets/img/blog/windows-on-digitalocean-droplet/configure_ip_gateway2.png" alt="configure ip" class="zoom-img" onclick="this.classList.toggle('zoomed')">

5. Set:
   - **IP address**
   - **Subnet mask**
   - **Default gateway**
   - **Preferred DNS server**
   - **Alternate DNS server**

   **Note**: You can keep the DNS same as shown in the SS. That is Google DNS IP 8.8.8.8 and alternate DNS as 8.8.4.4   

6. Click ok and save and wait. If the internet works great, otherwise try the same with other Ethernet interfaces.
7. Close the recovery Window.


---

<a name="part-7"></a>

## Part 7: Hurray, finally you have configured your Windows machine on droplet

- **Linux User**: Use Remmina Client 
  - Enter the IP address and Port `ip:port` which is configured for RDP on Windows Machine. Hit Enter.
  - Enter the Username, Password and Click OK and start using.

  <img src="/assets/img/blog/windows-on-digitalocean-droplet/login_using_remmina.png" alt="login using remmina" class="zoom-img" onclick="this.classList.toggle('zoomed')">

- **Windows User**: Use **Remote Desktop Connection**
  - Enter the IP address and Port `ip:port` which you configured for RDP on your Windows Machine.
  - Enter the username. Click Connect.
  - Enter the Password.
  
  <img src="/assets/img/blog/windows-on-digitalocean-droplet/login_using_rdp.png" alt="login using rdp" class="zoom-img" onclick="this.classList.toggle('zoomed')">

Start using your Windows on the droplet VM2. You can preserve that **windows10.gz** file for later use and you can remove the droplet VM1 which you created in Part 1 for preparing Windows Disk.

---

<a name="part-8"></a>

## Part 8: Allocate Unused Storage (Optional)
- Open **Create and format hard disk partitions** from Start menu.
- Create the disk out of unallocated space (if any).



<!-- ## Equations

This theme supports rendering beautiful math in inline and display modes using [MathJax 3](https://www.mathjax.org/) engine.
You just need to surround your math expression with `$$`, like `$$ E = mc^2 $$`.
If you leave it inside a paragraph, it will produce an inline expression, just like $$ E = mc^2 $$.

To use display mode, again surround your expression with `$$` and place it as a separate paragraph.
Here is an example:

$$
\left( \sum_{k=1}^n a_k b_k \right)^2 \leq \left( \sum_{k=1}^n a_k^2 \right) \left( \sum_{k=1}^n b_k^2 \right)
$$

Note that MathJax 3 is [a major re-write of MathJax](https://docs.mathjax.org/en/latest/upgrading/whats-new-3.0.html) that brought a significant improvement to the loading and rendering speed, which is now [on par with KaTeX](http://www.intmath.com/cg5/katex-mathjax-comparison.php).

---

## Citations

Citations are then used in the article body with the `<d-cite>` tag.
The key attribute is a reference to the id provided in the bibliography.
The key attribute can take multiple ids, separated by commas.

The citation is presented inline like this: <d-cite key="gregor2015draw"></d-cite> (a number that displays more information on hover).
If you have an appendix, a bibliography is automatically created and populated in it.

Distill chose a numerical inline citation style to improve readability of citation dense articles and because many of the benefits of longer citations are obviated by displaying more information on hover.
However, we consider it good style to mention author last names if you discuss something at length and it fits into the flow well — the authors are human and it’s nice for them to have the community associate them with their work.

---

## Footnotes

Just wrap the text you would like to show up in a footnote in a `<d-footnote>` tag.
The number of the footnote will be automatically generated.<d-footnote>This will become a hoverable footnote.</d-footnote>

---

## Code Blocks

Syntax highlighting is provided within `<d-code>` tags.
An example of inline code snippets: `<d-code language="html">let x = 10;</d-code>`.
For larger blocks of code, add a `block` attribute:

<d-code block language="javascript">
  var x = 25;
  function(x) {
    return x * x;
  }
</d-code>

**Note:** `<d-code>` blocks do not look good in the dark mode.
You can always use the default code-highlight using the `highlight` liquid tag:

{% highlight javascript %}
var x = 25;
function(x) {
return x \* x;
}
{% endhighlight %}

---

## Interactive Plots

You can add interative plots using plotly + iframes :framed_picture:

<div class="l-page">
  <iframe src="{{ '/assets/plotly/demo.html' | relative_url }}" frameborder='0' scrolling='no' height="500px" width="100%" style="border: 1px dashed grey;"></iframe>
</div>

The plot must be generated separately and saved into an HTML file.
To generate the plot that you see above, you can use the following code snippet:

{% highlight python %}
import pandas as pd
import plotly.express as px
df = pd.read_csv(
'https://raw.githubusercontent.com/plotly/datasets/master/earthquakes-23k.csv'
)
fig = px.density_mapbox(
df,
lat='Latitude',
lon='Longitude',
z='Magnitude',
radius=10,
center=dict(lat=0, lon=180),
zoom=0,
mapbox_style="stamen-terrain",
)
fig.show()
fig.write_html('assets/plotly/demo.html')
{% endhighlight %}

---

## Details boxes

Details boxes are collapsible boxes which hide additional information from the user. They can be added with the `details` liquid tag:

{% details Click here to know more %}
Additional details, where math $$ 2x - 1 $$ and `code` is rendered correctly.
{% enddetails %}

---

## Layouts

The main text column is referred to as the body.
It is the assumed layout of any direct descendants of the `d-article` element.

<div class="fake-img l-body">
  <p>.l-body</p>
</div>

For images you want to display a little larger, try `.l-page`:

<div class="fake-img l-page">
  <p>.l-page</p>
</div>

All of these have an outset variant if you want to poke out from the body text a little bit.
For instance:

<div class="fake-img l-body-outset">
  <p>.l-body-outset</p>
</div>

<div class="fake-img l-page-outset">
  <p>.l-page-outset</p>
</div>

Occasionally you’ll want to use the full browser width.
For this, use `.l-screen`.
You can also inset the element a little from the edge of the browser by using the inset variant.

<div class="fake-img l-screen">
  <p>.l-screen</p>
</div>
<div class="fake-img l-screen-inset">
  <p>.l-screen-inset</p>
</div>

The final layout is for marginalia, asides, and footnotes.
It does not interrupt the normal flow of `.l-body` sized text except on mobile screen sizes.

<div class="fake-img l-gutter">
  <p>.l-gutter</p>
</div>

---

## Other Typography?

Emphasis, aka italics, with _asterisks_ (`*asterisks*`) or _underscores_ (`_underscores_`).

Strong emphasis, aka bold, with **asterisks** or **underscores**.

Combined emphasis with **asterisks and _underscores_**.

Strikethrough uses two tildes. ~~Scratch this.~~

1. First ordered list item
2. Another item
   ⋅⋅\* Unordered sub-list.
3. Actual numbers don't matter, just that it's a number
   ⋅⋅1. Ordered sub-list
4. And another item.

⋅⋅⋅You can have properly indented paragraphs within list items. Notice the blank line above, and the leading spaces (at least one, but we'll use three here to also align the raw Markdown).

⋅⋅⋅To have a line break without a paragraph, you will need to use two trailing spaces.⋅⋅
⋅⋅⋅Note that this line is separate, but within the same paragraph.⋅⋅
⋅⋅⋅(This is contrary to the typical GFM line break behaviour, where trailing spaces are not required.)

- Unordered list can use asterisks

* Or minuses

- Or pluses

[I'm an inline-style link](https://www.google.com)

[I'm an inline-style link with title](https://www.google.com "Google's Homepage")

[I'm a reference-style link][Arbitrary case-insensitive reference text]

[You can use numbers for reference-style link definitions][1]

Or leave it empty and use the [link text itself].

URLs and URLs in angle brackets will automatically get turned into links.
http://www.example.com or <http://www.example.com> and sometimes
example.com (but not on Github, for example).

Some text to show that the reference links can follow later.

[arbitrary case-insensitive reference text]: https://www.mozilla.org
[1]: http://slashdot.org
[link text itself]: http://www.reddit.com

Here's our logo (hover to see the title text):

Inline-style:
![alt text](https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 1")

Reference-style:
![alt text][logo]

[logo]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 2"

Inline `code` has `back-ticks around` it.

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```

```python
s = "Python syntax highlighting"
print s
```

```
No language indicated, so no syntax highlighting.
But let's throw in a <b>tag</b>.
```

Colons can be used to align columns.

| Tables        |      Are      |  Cool |
| ------------- | :-----------: | ----: |
| col 3 is      | right-aligned | $1600 |
| col 2 is      |   centered    |   $12 |
| zebra stripes |   are neat    |    $1 |

There must be at least 3 dashes separating each header cell.
The outer pipes (|) are optional, and you don't need to make the
raw Markdown line up prettily. You can also use inline Markdown.

| Markdown | Less      | Pretty     |
| -------- | --------- | ---------- |
| _Still_  | `renders` | **nicely** |
| 1        | 2         | 3          |

> Blockquotes are very handy in email to emulate reply text.
> This line is part of the same quote.

Quote break.

> This is a very long line that will still be quoted properly when it wraps. Oh boy let's keep writing to make sure this is long enough to actually wrap for everyone. Oh, you can _put_ **Markdown** into a blockquote.

Here's a line for us to start with.

This line is separated from the one above by two newlines, so it will be a _separate paragraph_.

This line is also a separate paragraph, but...
This line is only separated by a single newline, so it's a separate line in the _same paragraph_. -->
