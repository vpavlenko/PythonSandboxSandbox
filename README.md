PythonSandboxSandbox
====================

A sandbox-within-a-sandbox for Python #inception

Created in Dec 2013 by Philip Guo (philip@pgbovine.net)

This sandbox is suitable for running untrusted Python code over the Web. It enforces:
- limited CPU and wall clock time
- limited memory usage
- no file opening, reading, or writing
- no file permission changes
- no network access


It's been tested so far on a 64-bit Linux 3.4 distro (Amazon Linux on EC2).

Prerequisites:
- some modern Linux
- root access
- basic compiler toolchain (gcc, make, etc.)

Disclaimer: I am not a security expert, so use at your own risk. In particular, the setup
I describe does not do a chroot. Although `safeexec` can technically chroot, I haven't figured
out all the magical incantations required to get it working with my Python installation.

---

This doc highlights how I'm executing untrusted Python code in a double sandbox on Amazon EC2 with Apache CGI.

## Setup

Dec 2013 - I first installed the Apache Webserver with CGI support on an EC2 instance running Amazon Linux,
roughly following
[these instructions](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-LAMP.html) (without MySQL):

    # install Apache with CGI
    sudo yum update
    sudo yum groupinstall -y "Web Server" "PHP Support"

    # start Apache
    sudo service httpd start

    # configure permissions for a www group
    sudo groupadd www
    sudo usermod -a -G www ec2-user
    <log out of EC2 and then log back in>
    sudo chown -R root:www /var/www
    sudo chmod 2775 /var/www
    find /var/www -type d -exec sudo chmod 2775 {} +
    find /var/www -type f -exec sudo chmod 0664 {} +


I also compiled Python 3.3.3 from source and did `sudo make install` since I need to run Python 3.
The Python 3 executable is in `/usr/local/bin/python3`.


### Testing CGI

After installation, I was able to get this "hello world" Python 3 CGI script running as `/var/www/cgi-bin/hello.py`

    #!/usr/local/bin/python3
    print("Content-type: text/plain; charset=iso-8859-1\n")
    print("hello")
    import sys
    print(sys.version)

by visiting this URL:

    http://<EC2 instance URL>/cgi-bin/hello.py

Okay, so far so good. Now at least I can run Python code sent over the Web. Of course, there's no sandbox yet,
so if I execute malicious code, then my entire VM can get trashed.


## Sandbox 1: safeexec

I use `safeexec`, a generic process-level sandbox for Linux, as the first layer of protection.

I cloned `safeexec` from the [cemc version](https://github.com/cemc/safeexec) and added it as a
"fake Git submodule" using [this technique](http://debuggable.com/posts/git-fake-submodules:4b563ee4-f3cc-4061-967e-0e48cbdd56cb):

    git clone https://github.com/cemc/safeexec.git
    # the trailing '/' is VERY important so that Git doesn't treat as submodule, eek
    git add safeexec/

I modified this version by adding an explicit `--uid` option, which lets me execute
a child process as a specified UID. This was required for me to get Python subprocesses
to play nicely with Apache CGI. (The default `safeexec` behavior is to execute a child
process with UID = PID, to ensure that all child processes are unique. [Read here](safeexec/README)).

To make `safeexec` available to Apache CGI, I compiled it, copied the binary to `/var/www/cgi-bin`,
and set the proper ownership and permission bits:

    cd safeexec/
    make
    # hopefully it succeeds
    cp safeexec /var/www/cgi-bin/
    cd /var/www/cgi-bin/
    # set these magic bits, or else the sandbox won't work!
    sudo chown root:root safeexec
    sudo chmod u+s safeexec


### Blocking network access

By default, `safeexec` executes child processes with a group ID (gid) of 1000.
Run this magic `iptables` incantation to block all network access for gid=1000,
which will prevent sandboxed programs from accessing the network:

    sudo /sbin/iptables -A OUTPUT -m owner --gid-owner 1000 -j DROP


### Testing the sandbox locally

First `cd /var/www/cgi-bin`, since we will be working with files in there.

Here is how to invoke the sandbox with Python:

    ./safeexec --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c "print('hello world')"

Let's break this down:
- `--cpu 6 --clock 4` limits the child process to 6 CPU seconds and 4 wall clock seconds, respectively.
- `--mem 250000` limits to ~250MB of memory. If you set this number too low (e.g., below 30000), then Python won't have enough memory to start up.
- `--uid 99` sets the UID to the `nobody` user. On my EC2 instance, `nobody` has a UID of 99. Check `/etc/passwd` for yours. This is important to both prevent the child process from tampering with your files, and also to get Python subprocesses working. If you *don't* explicitly set the UID, then `safeexec` will pick a "random" fake UID for improved isolation, and Python will sometimes fail to initialize, since its process UID isn't a real user. (This was an aggravating bug to hunt down, ergh!!!)
- `--exec /usr/local/bin/python3 -c "print('hello world')"` executes a Python string

If all goes well, the program should successfully run and terminate with the following output:

    hello world
    OK
    elapsed time: 0 seconds
    memory usage: 0 kbytes
    cpu usage: 0.036 seconds


#### Limiting running time

Let's try to run an infinite loop:

    ./safeexec --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c "while True: print('argh')"

It should die after 4 seconds with a `Time Limit Exceeded` error. Cool!


#### Limiting memory

Now let's try a memory bomb (note that I set `--mem` to a smaller value so it will die sooner):

    ./safeexec --mem 50000 --uid 99 --exec /usr/local/bin/python3 -c '
    x = 2
    while True:
        x = x * x
    '

It should die with a `MemoryError` within a second or so. Cool^2!


#### Creating, reading, and writing files

Now let's try to create and write to a file:

    ./safeexec --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c \
    "f=open('blah.txt','w');f.write('hi');f.close()"

It should fail with a `PermissionError`. Note that if you run without the sandbox, it should work:

    /usr/local/bin/python3 -c "f=open('blah.txt','w');f.write('hi');f.close()"

Okay, keep that `blah.txt` file there and re-run the sandboxed command to try to overwrite an existing file:

    ./safeexec --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c \
    "f=open('blah.txt','w');f.write('hi');f.close()"

Should get permission denied again.

What about reading a file, like `/etc/passwd`?

    ./safeexec --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c \
    "print(open('/etc/passwd','r').read())"

Ah, interesting -- we can still read most world-readable files as the `nobody` user, so that's a problem.
But we will fix this later with our second sandbox layer :)


#### Changing file permissions

What about changing the permissions on a file or directory? If this is allowed, then an attacker can
force a Denial-of-Service by making your website files inaccessible to the public. Eeek!

Let's assume that `blah.txt` still exists. First let's set its permission to 0 (without sandboxing) and verify that it worked:
    
    ls -l blah.txt # should see regular permissions, e.g.,: -rw-rw-r-- 1 ec2-user www 2 Dec 28 16:18 blah.txt
    /usr/local/bin/python3 -c "import os; os.chmod('blah.txt', 0)"
    ls -l blah.txt # should see NO permissions,      e.g.,: ---------- 1 ec2-user www 2 Dec 28 16:18 blah.txt
    chmod 664 blah.txt # restore to normal
    ls -l blah.txt # should see regular permissions, e.g.,: -rw-rw-r-- 1 ec2-user www 2 Dec 28 16:18 blah.txt

Okay, now let's try to change permissions from a sandboxed process:

    ./safeexec --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c \
    "import os; os.chmod('blah.txt', 0)"

This should result in a `PermissionError`.


#### Blocking network accesses

Let's first try running a simple Python script that fetches the contents of the Python home page:

    /usr/local/bin/python3 -c \
    "import urllib.request; print(urllib.request.urlopen('http://python.org/').read())"
    
This should print out some HTML fetched over the network.

Let's run it in the sandbox:

    ./safeexec --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c \
    "import urllib.request; print(urllib.request.urlopen('http://python.org/').read())"

You should now see an error, since `iptables` blocked network accesses for gid=1000, which the child process was running as.

To confirm, run `safeexec` again with `--gid 2000` (or anything besides the default of 1000), and it
should be able to access the network:

    ./safeexec --gid 2000 --cpu 6 --clock 4 --mem 250000 --uid 99 --exec /usr/local/bin/python3 -c \
    "import urllib.request; print(urllib.request.urlopen('http://python.org/').read())"


### Testing the sandbox on the Web via CGI

Now let's write a CGI script that allows the user to input arbitrary Python code and have it execute
within this sandbox. For now, let's 

```
#!/usr/local/bin/python3
import cgi
import os
import subprocess

# for debugging
import cgitb
cgitb.enable()

print("Content-type: text/plain; charset=iso-8859-1\n") # proper header

form = cgi.FieldStorage()
script = form['user_script'].value

args = ['./safeexec',
        '--cpu', '6',
        '--clock', '4',
        '--mem', '250000',
        '--uid', '99', # essential or else Python won't start up properly!!!
        '--exec']

args += ['/usr/local/bin/python3', '-c', script]

p = subprocess.Popen(args, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
out, err = p.communicate()

print('stdout:')
print(out)
print('stderr:')
print(err)
```

Save this file as `/var/www/cgi-bin/run_code.py`, make it executable, and visit:

    http://<EC2 instance URL>/cgi-bin/run_code.py?user_script=print("hello CGI!")

You should see something like:

```
stdout:
b'hello CGI!\n'
stderr:
b'OK\nelapsed time: 0 seconds\nmemory usage: 0 kbytes\ncpu usage: 0.024 seconds\n'
```

Okay, cool, now we're done with the first sandbox layer. But we must go deeper ...


## Sandbox 2: Pure-Python sandbox

The main pesky part about the existing `safeexec` sandbox is that the child process can still open
and **read** lots of files. `safeexec` provides two mechanisms to prevent file reading -- chroot
and limiting open file descriptors to zero -- but both are problematic for my use case. First, chroot
is really awkward and cumbersome, and leads to sysadmin headaches. Second, I can't simply limit open
file descriptors to zero, since Python itself needs to read a lot of files when it starts up.

So I've implemented a second layer of sandboxing, this one specialized for Python scripts. See
[`simple_pysandbox.py`](simple_pysandbox.py) for its code.

