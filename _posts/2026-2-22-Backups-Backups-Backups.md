---
title: Backups, Backups, Backups!
date: 2026-2-22 13:26:00 
categories: [homelab,selfhosted,professional-development]
tags: [servers,selfhosted]
---

## Waking up from the backup matrix

How did I ever live without a backup solution? That's a bad question, actually. It's pretty easy to live without a backup solution. The real question is why did I ever live without a backup solution?

For so long I buried my head in the sand when it came to backups. It was easier a year or so ago; all I had was a simple Unraid all-in-one solution. I was running quite a few docker containers and a single VM, but Unraid handled snapshots for me, and it was all integrated into my NAS already-- I saw no need to freak out about backups... Until I had a few wake-up calls. The first was when I decided to migrate to Proxmox. I needed to take a mirrored cache pool and consolidate it into a single disk in Unraid so I could fit a second drive for Proxmox within the same server. I needed to take two NVME's and move them both to a single NVME drive. This was not a simple task (or maybe I myself made it into a non-simple task) primarily due to the nature of the file system. I needed to move the information off of the cache disks to a new place, then move it back to the new disk. To do this, I had the ingenious idea to simply move it to my array... which corrupted several databases and completely wiped my installed containers. My saving grace was Unraid's Community Apps feature, which allowed me to reinstall the containers I had with the settings that Unraid remembered, preventing me from needing to start from scratch. This took me almost 5 or more hours to recover from, and some metadata is still missing from my media front-end, Plex.

The second wake-up call was when I almost lost all of my photos from the past 10 or so years. In true self-hoster fashion, I've long since sworn off anything like iCloud backups, Google Photos, or any other sort of storage solution owned by a major corporation. This is a great idea for anyone who is serious about backing up data, and I was not. While fixing an issue with an Immich docker container, I accidentally deleted my entire photo library from Immich. I was convinced that I had just lost everything until I remembered that only a week prior, I had decided to back up my entire photo library to a single USB stick. Talk about insane foresight! Can you tell I break stuff often?

A thought had struck me several times throughout the process: "does this need to be so hard?" Good news, past me, it doesn't!

## Getting it into gear

My backup journey started with the aforementioned Proxmox migration. I decided to set up a Proxmox cluster with a few Optiplex's and my NAS box (maybe I'll write about this later). After moving some services to separate VMs on separate boxes, I knew I needed some sort of consolidated backup solution that would safeguard myself against stupid mistakes or bad advice. I learned that Proxmox actually has a dedicated operating system for this very task: Proxmox Backup Server. This OS gets installed unto a dedicated box, separate from the Proxmox cluster. It then integrates directly with the Proxmox Visual Environment and allows you to perform deduplicated backups (and snapshots) for VMs. I ran this initially in a separate box with three 500GB drives I had laying around in a RAID5 configuration. This was fine for a few days until one of the drives began throwing a SMART error or two, which meant I now had a separate storage box to keep up with. This was not ideal for me, as I already have a NAS-- Why would I want two storage solution headaches to deal with? I also was looking into offsite backups at the time, and Proxmox Backup Server would require me to set up a server on-site and ANOTHER server off-site. This brings the total to... 7 machines that I am actively maintaining. Don't get me wrong, I like homelabbing, but I don't want to be jacked into some issue every waking second of my life. I needed to figure something else out.

## Enlightenment

After much research, I landed on Veeam. Veeam has a completely free server application called Veeam Backup & Recovery. I had wrongly assumed that this was a separate OS, but it actually runs on the Windows operating system only. Bummer. One Windows Server 2025 Install later, and I was on my way to never worrying about wiping my data again. The setup was not painless at all. The Veeam Backup & Recovery user interface is painfully antiquated, and lacks a web GUI in the free editions of the software. Brave has an ongoing issue (at the time of writing this) where every so often, the Proxmox web UI will glitch out the browser, causing me to completely shut down Brave and re-open it.

## Configuring Veeam

I very quickly was able to get Proxmox VE hooked into Veeam, with all of my VM's being discovered automatically. After configuring a daily backup, I was set on the Proxmox end! I now only had to worry about my endpoints. I only have two separate devices, my MacBook and my Windows machine. I wanted a system image level backup for both so that in the event I need to move either to a new box, I can do so easily with a bootable USB. This is an option on Windows, but not for Mac. File-level backup it is... To perform this step, I created what Veeam calls Protection Groups, and added the desired machines to a separate Windows group and a Mac group, in case I ever want to expand the amount of endpoints I have. After doing this, I was given the option to deploy Veeam agents to each device, which will "phone home" for a backup periodically at my desired schedule. This was a breeze on Windows, just pass Veeam an Admin login and you're good to go! On Mac, it was much more involved. I had to export a configuration profile, download Veeam ONLY from that package, then import a config. Not so bad, but still much more work. The docs are also not extremely clear on this matter, which led to a lot of mis-steps on my end.

After adding all of my devices and configuring my schedules, I simply needed to add my NAS as a backup repository, specify the folder, and perform the backups! This actually came in handy shortly after configuring Veeam, as I had to restore a VM to a previous state after butchering some config changes.

## "But what about off-site backups?", you ask!

I was hoping to backup to my Backblaze B2 bucket directly from Veeam, but this is not available on the free version of the software. This is completely okay in a post-rclone world, though! On Unraid, we can actually run a userscript using rclone to back up my desired repositories to my Backblaze B2 bucket weekly!

And just like that, we are fulfilling the golden rule for backups: 3-2-1.

### 3 copies of our data

One copy on the actual VM/NAS storage, one copy in Veeam, and one copy in Backblaze.

### On 2 different mediums

This is cheating, but still technically true. One digital copy on the NAS and one cloud copy via Backblaze (although I am looking into archival M-Discs in the future).

### 1 backup copied off-site

Backblaze fulfills this perfectly! I can even add a redundant bucket within the Backblaze console.

## Conclusion

Are there a few changes I'd like to make? Sure! I'd love to have more redundant copies of my data stored locally, like another NAS. I'd also like to upgrade my Veeam license to simplify reduce the points of failure. All in all, I think this is a great step for my homelab. Now the data will never die!
