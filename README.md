# Welcome to my Nitro Home Server Project

For a long thing I've wanted to repurpose an old Nitro 5 9th generation laptop that I no longer use due to getting a more specialized laptop (Macbook Air M1) for my needs and this year (2025) I decided to make the leap into exploring homelabbing so that I can become more functional and self-sufficent on the web (everything is turning into a subscription). This repo will document my journey for my own sake (just in case something breaks or I have to resetup any aspect of the homeserver) and so that it may be able to help someone else out there who is just like me but has no idea on how or where to get started.

### Considerations to bare in mind

 1. My nitro laptop has a broken screen that happened sometime when I was travelling internationally in my brief case so I had to come up with a solution that could work around that limitation and most importantly cheap (I'm very frugal)!
 2. I ultimately wanted to have my own NAS / deploy my own gallery website (because I am a photographer who consume massive amounts of data as the years go by aka. a Data Horder)

# Initial decisions I made
After doing some research, I learned of a software called proxmox which would act as the OS of my nitro 5 (who ill formally call Nitro going forward) to leverage the resources of my laptop.

# Proxmox Installation
Despite everything that I saw online about how I should run proxmox, the best solution is definitely bare-metal but I was not about to do that I was not able to navigate through the Proxmox installation without a functioning screen. I tried everything, from 

 1. Installing proxmox to my bootdrive from another laptop (did not work)
 2. Tried to install proxmox headless (since I had no screen) (could not make it through the install gui or terminal)
 3. Install proxmox through a USB but selecting the boot drive (could not make it through the install gui or terminal)
 
 I was frustrated, dejected, and ready to give up, this seemed like a task too big for me to overcome myself. I was just ready to bite the bullet and order a new Nitro screen from eBay and replace it myself when I decided to just try something for the fun of it and you would not believe me when I say that it actually worked!

## What actually worked for me
Installing Proxmox on a virtual machine (VMWARE Workstation 17 Pro) and allocating it as much resources as I could since the laptop was running Windows10 on nvme0 (boot drive) and then logging in from M1 and doing everything else from there (yipee!!)

I was so frustrated when it actually worked because everything I looked online spoke about potential issues of not having proxmox run on bare-metal or the limitations of device passthrough through a virtual machine but this taught me a really big lesson which is that **everyone's needs are not my needs**. And so I should take everything I learn on the internet with a grain of salt and just try it. Had I just done that from the beginning, I would not have almost given up on this hobby/dream of repurposing this laptop to now serve me!

#### So now we are off to actually realizing this dream

# My first proxmox LXC installation - Adguard Home
If its one thing you should know about me is that I am not a big fan of the state of the internet currently. Misinformation, AI Slop, decreasing attention spans, you name it, I don't even enjoy my time on social media anymore because I feel like I am always being pressured into thinking or feeling a kind of way and that is no more evident than with the amount of garbage ads there are EVERYTHING on the internet. 

Now I understand that ads are a necessary way of paying it forward for the content I consume but not the absolute cesspit that has plagued social media and the internet as of recently.

# Huawei HG8145X6-12 DNS Configuration for Adguard Home 
I am mainly making this tutorial because I basically trialed and errored my way into the right configuration but it took me alot longer that it should have (over 3 hours) and I want to make it easier for anyone who has the same modem/router model to figure it out quicker.

#### You're really only concerned with 2 tabs when you login to your router's admin panel which is:

 1. LAN -> DHCP Server Configuration - Change your Primary DNS Server to point to your Adguard Home IPv4 Address

![enter image description here](https://i.postimg.cc/8kh1jpMN/Screenshot-2025-10-10-at-12-29-49-AM.png)

 2. IPv6 -> DHCP LAN Address Configuration - Change DNS Information Setting, DNS Source on the LAN Side to **Static Configuration from DNS Agent** and fill in the Preferred DNS as your Adguard Home IPv6 Address

![enter image description here](https://i.postimg.cc/mZQbtLCx/Screenshot-2025-10-10-at-12-29-34-AM.png)


*Last updated 10/10/2025 - Hoping to keep this updated weekly as the home-server grows*
