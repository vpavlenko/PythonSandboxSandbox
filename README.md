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


### Sandbox 1: safeexec

I cloned `safeexec` from the [cemc version](https://github.com/cemc/safeexec) and added it as a
"fake Git submodule" using [this technique](http://debuggable.com/posts/git-fake-submodules:4b563ee4-f3cc-4061-967e-0e48cbdd56cb)

    git clone https://github.com/cemc/safeexec.git
    git add safeexec/

I modified this version by adding an explicit `--uid` option, which lets me execute
a child process as a specified UID. This was required for me to get Python subprocesses
to play nicely with Apache CGI. (The default `safeexec` behavior is to execute a child
process with UID = PID, to ensure that all child processes are unique. [Read here](safeexec/README)).
