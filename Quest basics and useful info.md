
# Quest basics, best practices for shared computing environments, bash scripts, and moving data from your computer to Quest

We will start with Quest basics!

## What is Quest?

The vast majority of software we will be using in this class needs to be run in a High-Performance Computing (HPC) environment. What does that mean? That means we will be running software that either requires more memory, more processors, or more time than your laptop can provide. Instead we will use Quest, Northwestern's HPC.

There is a fair amount of documentation for Quest online. 

[Quest quick start guide](https://kb.northwestern.edu/quest-quickstart)

[Quest user guide](https://kb.northwestern.edu/page.php?id=quest)

You might still come across somethings it doesn't answer (yet). You can ask us specific questions not covered by the documentation, or (if we don't know the answer) contact quest-help@northwestern.edu. There is also a Slack channel - [genomics-rcs.slack.com](genomics-rcs.slack.com) that can be accessed using your Northwestern email address.

## How do I access Quest?

In your Terminal or other command line window, you will type (with your netid):

-----
```
ssh -X <netid>@quest.it.northwestern.edu
```
-----
Quest will then prompt you for your password. After entering it, you should see something like this:

-----
```
Last login: Thu Jan  3 14:36:11 on ttys001
dhcp-10-105-178-246:~ elizabethmallott$ ssh -X ekm9460@quest.it.northwestern.edu
ekm9460@quest.it.northwestern.edu's password: 
Warning: untrusted X11 forwarding setup failed: xauth key data not generated
Last login: Wed Dec 19 14:17:10 2018 from dhcp-10-105-138-84.wireless.northwestern.private
Quest has three partitions, which you can specify in your script (for example:  #MSUB -l partition=quest6).

Partitions and node specifications are below:
- quest5: 24 cores & 128 GB memory per node
- quest6: 28 cores & 128 GB memory per node
- quest8: 28 cores & 96 GB memory per node

Users cannot request more than 4 cores and/or 4 GB of memory in total for all the applications 
they are running on a login node

If you would like to purchase nodes or storage for dedicated usage, contact quest-help@northwestern.edu.

Please remember that Quest is not approved for contractually or legally restricted data.
Ensure to anonymize or de-identify your data before moving to data to quest.
Contact us if you need a computing environment for the use of restricted data.

Information on Quest can be found at http://www.it.northwestern.edu/research/user-services/quest/index.html.
For any questions, please contact us at quest-help@northwestern.edu.
[ekm9460@quser12 ~]$ 
```
-----

You will be in your home directory, by default. From here, you can navigate to any directory to which you have access.

Which brings us too...

## Best practices for a shared computing environment

Quest is not only super powerful, it's also used by 100s (maybe 1000s?) of people on campus. Therefore, it's in everyone's best interest to follow some general guidelines to not mess up anyone else's work.

1. Do not move or delete resources in a shared folder

  We will all have access to a shared allocation, e30740, for this class. As part of this allocation, there is a shared projects folder, /projects/e30740, where necessary software and reference databases will live. DO NOT DELETE OR MOVE THESE.
  
  Everyone should make their own folder *within* /projects/e30740 from which to run your code or store data. Let's do that now.
  
  -----
  ```
  cd /projects/e30740
  mkdir emallott
  ls
  ```
  -----
  
  You should see your folder listed in the directory.


2. Do store your scripts and text files in your home directory, /home/<netid>. Feel free to store any you-specific resources here as well, but be aware that there is a 80 GB size limit on this directory. It is backed up, though, which is useful.

  As an example, this is what my home directory looks like (ah! a mess!)
  

3. Do not run software or jobs from the head node or login node. This will make others using the HPC unhappy and likely get you an email from Research Computing Services scolding you.

   You will either need to start an interactive job, for shorter jobs (these will only continue running while your computer is open).
   
   -----
   ```
   msub -I -l nodes=1:ppn=4 -l walltime=01:00:00 -q short -A e30740
   ```
   -----
   
   Or run a bash script (stay tuned)
   

4. Request appropriate resources and the appropriate queue for your jobs. If you don't, you risk having your job blocked, canceled for you by Research Computing Services, and/or finish without completing. This bit will make more sense soon, and we will also help you figure out what "appropriate" is.

Liz, you have just said a lot of things I don't understand...

## Bash scripts and specifying resources

When we are running jobs on Quest, we usually use a bash script. This is a short bit of code that specifies: 
1. What your job is called
2. Which allocation to charge (for this class, always e30740)
3. What updates to email you
4. How many resources we need (amount of memory, number of processors, amount of time)
5. Whether or not to create an error and output log (which you should always do)
6. Where to run the job (the queue)
7. Which software to load


Here is an example of an old bash script, otupick.sh, used to run QIIME 1:

-----
```
#!/bin/bash

#MSUB -N otupick
#MSUB -A e30740
#MSUB -m abe
#MSUB -M elizabeth.mallott@northwestern.edu
#MSUB -l nodes=1:ppn=4
#MSUB -l walltime=1200:00:00
#MSUB -j oe
#MSUB -q normal

module load python

pick_open_reference_otus.py -i /home/ekm9460/seqs.fna -o /home/ekm9460/0718_output -m sortmerna_sumaclust -a -O 4
```
-----

Bash scripts are created in text files (remember nano from Katie's command line tutorial) and become executable files by using the chmod command. 

You submit the job like this on Quest from whichever directory in which the script lives (usually your home directory).

-----
```
msub otupick.sh
```
-----

You can also create loops within your submission script to submit jobs for multiple files simultaneously. But we will talk more about that when we get to shotgun analysis.

## Cyberduck

One last useful thing to know about Quest is how to move data onto and off of it. Katie talked a little about commands used for downloading from FTP servers or websites and is going to cover getting data from public repositories in a minute. But you might find it useful to move data between your computer and Quest.

You can use the command line to do this, but an easier and more intuitive way is to use what we call a Secure File Transfer Protocol Client (SFTP). We will use Cyberduck (https://cyberduck.io/) for this.

Once you have installed Cyberduck, you can use the following steps to connect to Quest:

1. Click Open Connection in the upper left of the Cyberduck window
2. At the top of the Open Connection window that appears, Select SFTP (SSH File Transfer Protocol) from the drop-down menu.
3. Enter quest.it.northwestern.edu for server specification
4. Enter your NetID in the Username: box and leave the Password: box empty to prevent your NetID password from being saved in a file on your personal computer. Public Key Authentication is not supported.
5. Click Connect. You will see a Login failed window.
6. Enter your NetID password in the Password: field.
7. Click Login.

Then you will see your home folder. You can navigate this just like you would the filesystem on your own computer. You can drag and drop files into it or use the File Menu at the top to upload and download files. It has the added benefit of a more intuitive interface for visualizing a mess of files, like I have created!

So! That's a basic overview of Quest. We will be going through all of this again and you will experience it hands on as we have you work through the tutorials in class over the next 4-5 weeks. Feel free to ask questions at any time and spend some time looking at the online Quest resources.
