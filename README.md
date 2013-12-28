PythonSandboxSandbox
====================

A sandbox-within-a-sandbox for Python #inception

Created in Dec 2013 by Philip Guo (philip@pgbovine.net)

This sandbox is suitable for running untrusted Python code over the Web. It enforces:
- no file opening, reading, or writing
- no file permission changes
- no network access
- limited CPU and wall clock time
- limited memory usage

It's been tested so far on a 64-bit Linux 3.4 distro (Amazon Linux on EC2).

Prerequisites:
- some modern Linux
- root access
- gcc

Disclaimer: I am not a security expert, so use at your own risk. In particular, the setup
I describe does not do a chroot. Although `safeexec` can technically chroot, I haven't figured
out all the magical incantations required to get it working with my Python installation.

---

This doc highlights how I'm executing untrusted Python code in a double sandbox on Amazon EC2 with Apache CGI.

## Setup

Dec 2013 - I first installed Apache with CGI support on an EC2 instance running Amazon Linux, roughly following
[these instructions](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-LAMP.html) (without MySQL):

Install Apache:
    sudo yum groupinstall -y "Web Server" "PHP Support"

Start Apache:
    sudo service httpd start

configure permissions:
    sudo groupadd www
    sudo usermod -a -G www ec2-user
    <log out and log back in>
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

    http://<EC2 URL>/cgi-bin/hello.py

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
