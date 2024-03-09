---
title: "Babuk Nightmare"
date: 2023-11-01T07:07:07+01:00 
---

Last week one of our vmware esxi servers stoped working.At first I notinced that my password is not working.   
I put the server into the rescue mode and boot it into [RescueCd](https://www.system-rescue.org/).   
After mounting VMFS partitions I saw that `vmdk` files renamed to `vmdk.babyk` and on that moment I knew why my password not working,


## I was hacked!


Alongside to babyk files, there was a file named "How To Restore Your Files.txt":

```txt
Your encrypted ID:
55b5b4d9-97fe-43dc-a105-b0bd15e00cd4

Please pay 100,000$ in Monero to the following account:
-----------------------------------------
88PaUKVy7YaRbhyw1spSoF402QgH98z589juZAkQCskX9sncUCSVhCCZMb7J5grAdmPyi5WvH1Ub73xXTKzM7e3G9p8ESd4
-----------------------------------------

Please contact the email address below within 3 days and pay the ransom.
Note: If you do not receive an email in your inbox during communication, please check whether it is in spam.
-----------------------------------------
ablyteqotg@tutanota.com
-----------------------------------------

Warning: Please do not attempt to decrypt privately or make any changes to the server, otherwise we will no longer be responsible for the losses caused by this incidence
```

So now it's obviuse that my data is encrypted and the only way to restore my data was to pay $100,000.   
I had 2TB + 2 x 12TiB HDDs on this server but I had just one thing among my data which I care about.   
There was a 12TiB disk that I running a [SeaweedFS](https://github.com/seaweedfs/seaweedfs) instance on it.    
I use RDM to connect that pysical disk to my virtual machine for better performance.

Immedetly I begun an negotition with the hacker and there is no luck. He demending $100,000 which I couldn't afford.

After disappointing about the hacker, I began to research about the ransomware. After the first search I hit to a [vmware.com page](https://blogs.vmware.com/security/2022/10/esxi-targeting-ransomware-tactics-and-techniques-part-2.html) with a good info about it.

![Vmware info about Babuk](/images/babuk/vmware-info-babuk.png)

Its turns out it's name is `Babuk` and [it's source code](https://github.com/Hildaboo/BabukRansomwareSourceCode/tree/main/esxi) is leaked. Suddenly some light of hopes begin to shine!

In that github repository I found `ext/dec/main.cpp`.   
So basiclly it has two function.

	- void find_files_recursive(char* begin)
	- void decrypt_file(char *path)

the names are self-explainery.    
I not caring about find_files_recursive and obvusly all of the decrypting done by `decrypt_file`.

So it's read 32 bytes at the end of encrypted file and with using `m_priv` global variable and `curve25519` algo, some shared key will create.   
With that shared key, decrypting was easy, I could my files back!

```c
void decrypt_file(char* path) {
  ...
  FILE *fp_key = fopen(path, "r+b")
  if(fseek(fp_key, -32, SEEK_END) == 0) {
    fread(u_publ, 1, 32, fp_key);
    curve25519_donna(u_secr, m_priv, u_publ);
  }
  fclose(fp_key);
  truncate64(path, fstat.st_size - 32);
  FILE *fp = fopen(path, "r+b")
  if(uint8_t* f_data = (uint8_t*)malloc(CONST_BLOCK)) {
    do {
      wholeReaded += readed = fread(f_data, 1, CONST_BLOCK, fp);
      if(readed) {
        sosemanuk_encrypt(&rc, f_data, f_data, readed);
        fseek(fp, -readed, SEEK_CUR);
        fwrite(f_data, 1, readed, fp);
      } else break;
    } while(wholeReaded < 0x20000000 && wholeReaded < (fstat.st_size - 32));
  }
  ...
}
```

I already heard about `curve25519` but never really used it, I searched and read about it.   
Even in those moment of losing 12TiB of data, I enjoyed the algo.

## X25519
Here is the summary:   
Mr evil hacker create a private key for himself from a random stream like /dev/urandom and save it into `hacker_private` variable.
After that he call curve25519_donna with that private key and generate a public key and save it into `hacker_public` variable.

When the Mr evil hacker attack to my server, for each file he does these steps:
1. He creates a private key using a random stream as name that `file_private`.
2. He pass that private key to `curve25519` and generate a public key `file_public`.
3. Using curve25519 with his `hacker_public` and `file_private` creates a shared key names `file_shared`.
4. Instancely he earse `file_private` from memory.
5. Encrypt the file using `file_shared` key.
6. Append `file_public` at the end of the file to persist it.

After the evil received his ransom payment, he gives you a executable file containing his `hacker_private` key. This file does these steps to decrypt your files:
1. Read 32 bytes of end the file and put it into `file_public` variable and truncate those bytes from your file.
2. using `hacker_private` and `file_public` creates `file_shared`.
3. Decrypt your file.


In that moment of excitment suddenly I felt hopeless because for decrypting my files I need Mr evil private key and he doesn't coopertive!


Hours latter when I showing the `Babuk` encoder source code to my college, I saw a miracle.
I found a condition!   
In the condition hacker because of large size of vmdk files, limits his program to first 500MB of the disk.

![Spongebob Squarepants Rainbow](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExZmliYzQ0MTNpcmZmbmJ0emp0bTFoNWtwbHd0c2hzcDN1OTNlbnh2NSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/SKGo6OYe24EBG/giphy.gif)

Immedetly I ran fdisk to see what is going on my disk.

As I expected there is no parition table. Those kind of metadata stored on top the block storage and when 500MB of your disk encryped you cannot expact to these things work normally.

I still does not sure about the ransomware. Maybe the hacker modified the source code and removed that condition or raise the limit to a point that my data was not recoverable.   
So in that rescue OS I installed `hexcurse` and start to traviase 12TiB of binary data.   
After few moment of jebirush I found text and readable data!   
That was a log file, part of [Certbot](https://github.com/certbot/certbot).

![Certbot Log](/images/babuk/certbot-log.jpg)

So now I was certen that big part of my data was there but there is no partition table, there is no filesystem.

I knew there is a lot of hours trying recovering my data and no-sleep nights but I did what I should to do.

## Safety First

So before anything I schucle a pysical access time with datacenter and headed to the datacenter.
I depart both of my 12TiB hdds and came back to the office.
Since these drives are in 3.5 inch form, I turn on my very old friend, A HP desktop computer with 4 sata 3 jack and enough supply power for both of those drives.

![Two HDDs](/images/babuk/two-hdds.jpg)

I download and mount debian 12 live iso into a usb stick and boot up the PC.
using dd I backup my important drive to another drive
AND IT TOOK 30 HOURS TO COMPELETE.

After that i power off the PC and disconnect the original disk and boot debian up again.

## Partition Table Recovery

So first thing first, I had to understand every level of storage to begin the process.
ESXi saving data to the disk (RDM) directly but why when I access to VMFS volume it has two file?

After a quick search i notinced that for every virtual disk, esxi creates a file descriptor containing some metadata about that vritual disk.   
Actually first vmdk is the descriptor. mine was completely burnt. it's file size was less than 512MiB and there is no way to restore it.
Based on the file size second file was a pointer to pysical drive.   
But How Esxi save the data in this rmdp file? Is there any custom layout?   
So With a quick search I released the answer is NO.
If your .vmdk contains `-rdm` or `-flat` your virtual disk is raw image of disk.

So my backup drive is bascally is broken drive that first 500MiB of top it is missing, including partition table.

Because I didn't have any data about count and location of partitions I couldn't just create partition table using fdisk, If I had it would much easier.

So after spending some hours searching I stop unon a utility named "testdisk".
This tool has a command to analyze the disk and find find paritions using their filesystem fingerprint, It's genuis.
So I begin to analyze the backup disk and after few minutes, vooolaaa! It found them!!

![Testdisk analyze result](/images/babuk/testdisk-analyze-result.png)


## Ext4 Recovery

So I gone a step forward, now I have partition table.
First partition was /boot filesystem with no important data. You can always rebuild your /boot partition.
The third was a swap partition and again no need to worry about.
But my whole data was in the second one and becuase some inital and critical part of its filesystem was in the first 500MiB it was damaged.


So to be more specific, in very first blocks of every ext4 filesystem there are few things named `superblock`, `group descriptors` and `bit masks`. basiclly these blocks are holding information about the whole other blocks, inodes and data in filesystem... like a map.   
The good news is ext4 backup these precious superblocks mutiple places in the partition so you can restore them if you lost the primary one.   
Bad news is i didn't have superblock backup locations, I mean who does?!   
Of course you can calculate them if you have "logical block size" and again I wasn't sure!   
You should know at this point my server was down about 48 hours and I didn't have time to start over.

So agian I back to my new friend "testdisk" and it didn't let me down.
Using "Superblock recovery" feature I was able to locate all my superblock copies.

![Locating superblocks using Testdisk](/images/babuk/testdisk-superblocks.png)

As you can see, testdisk suggested that by running `fsck` you can begin to fix the filesystem using a superblock backup.
But it didn't mention for repairing a 12TiB parition how much memory you will need and hosntly I never knew!
Every time i ran this command after few moments it'll killed by kernel for using too much memory.

I found a alternative: `e2fsck`   
If you make a configuration file named `/etc/e2fsck.conf` and put a directory path for it's tempelory files it will use less memory.
```ini
[scratch_files]
directory = /var/cache/e2fsck
```
Still it needs 16 GiB of ram which my old PC nor have and support and despritely make very big swap parition.

![e2fsck memory usage](/images/babuk/e2fsck-memory-usage.jpg)

And it took 24 LOOOONG hours to finish with no progress bar.


But in the end It worth days of hardwork and i was able to access my files. Easy Peasy!

![Seaweedfs volume files](/images/babuk/seaweedfs-volume-files.jpg)


So if you ever need my help about this nasty ransomware please don't hasitate to contact me.