## TryHackMe Brute It Writeup
This isn't a particularly difficult room but I didn't see any writeups  
currently made for it, (or at least accepted) so I decided to make one.  
Also because I havent done any writeups before and this seemed like  
a good place to start.  

### Reconnaissance
First I started with a service scan with nmap  
![](nmapoutput.png)  
we see that ssh and apache are the only things running, this is a very  
simple web server. Now lets bruteforce the web directories with gobuster.  
![](gobusteroutput.png)  
The output we get shows a `/admin` directory that we can acess and also  
a `/server-status` directory that we cant access.   
![](loginform.png)  
Upon viewing the `/admin` we get a login portal. Lets also check the html.  
![](login-inspected.png)  
Upon viewing the source html we also see a comment addressed to a  
john letting us know that the login username is admin.   


### Getting User Access
Were going to have to bruteforce this login page, so lets make a more  
efficient wordlist. Since we know the password is going to be 6 characters  
long I filter the rockyou.txt wordlist to only contain passwords that are 6  
characters long. This will save us time on the actual bruteforcing  
process.  

`cat rockyou.txt | grep -x '.\{6,6\}' > 6rock.txt`  

Now, Im going to use OWASP ZAP to bruteforce this login form as I find  
working with hydra clunky and unreliable. If you arent familiar with  
OWASP ZAP you can check out the very good TryHackMe room on it [here](https://tryhackme.com/room/learnowaspzap)  

After you get ZAP running and intercepting traffic, make a test login, I  
used the credentials admin:secret. Then in zap find that login request and  
fuzz it.  
![](about2fuzz.png)  
then select the text that you want to be changed for every request, which  
is going to be the password input. Then add the altered rockyou wordlist  
as the payload.  
![](zap-fuzzer.png)  
It doesnt take too long for ZAP to find the password. Only 222 requests.  
Sort the results by "code" in descending order so that when theres a  
successful login its the first one.  
![](zap-fuzzd.png)  
Using the found credentials we can login to the admin page. Where we  
will find our web flag and an RSA Private Key for the user john.  
![](adminlogged.png)  
At first I didnt know how to crack this key with john, but a quick google  
search with for "crack Proc-Type: 4,ENCRYPTED DEK-Info: AES-128" gave  
me this [blog post](https://sql--injection.blogspot.com/p/decrypt-ssh-keys.html) which detailed the steps required.  
First we'll need a python script to convert this key to something readable  
by John the Ripper. As ssh2john didnt work with my jumbo version of john  
I had to download a python script that would.  
`curl https://raw.githubusercontent.com/stricture/hashstack-server-plugin-jtr/master/scrapers/sshng2john.py > sshng2john.py`  
then run the script with `python3 sshng2john.py id_rsa > crackme`  
Theres some garbage data thats placed on the first line, go ahead and  
manually delete it. The hash file to be used by john should only be one  
line long. Your final output should look similar to this.  
![](crackmeoutput.png)  
Then we can go ahead and crack this hash with John  
![](cracked.png)  
Now that we have our ssh user and password lets login.  
![](ssherror.png)  
I initially got an error but another quick google search got me the [answer](https://stackoverflow.com/questions/29933918/ssh-key-permissions-0644-for-id-rsa-pub-are-too-open-on-mac)  
I needed. I had to change the rsa.key permissions with  
`chmod 400 rsa.key`  
After changing the permissions I was able to login just fine and get the  
user flag.  
![](user.png)  

### Getting Root Access
The first thing I did was check for sudoable programs with  
`sudo -l` and from there I saw that we could run `cat` with root access.  
So then I just got the root flag with `sudo cat /root/root.txt`  
![](root.png)  
But we also need the root password. So lets get the linux hashes and crack  
them. `sudo cat /etc/shadow` and `sudo cat /etc/passwd` and save the  
outputs to files on your own machine (can just copy and paste from the  
terminal). I saved mine as `shadow` and `passwd` respectively.  Then  
prepare the hashes for john with  
`unshadow passwd shadow > linuxhash`  
and then we can crack with john  
![](roothash.png)  

### Conclusion
This was an easy but fun little room. It was the actually the first CTF that  
I've done where you've been required to crack linux user hashes which was  
nice.  