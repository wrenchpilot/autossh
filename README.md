# Usage
* To use it with sshpass you must set some environment variables

```bash
export AUTOSSH_PATH='sshpass'
export AUTOSSH_LOGFILE=/var/log/autossh/your_connection_log_file_path
export AUTOSSH_DEBUG=1
```

```bash
# Note `getpw()` is a local function to pull password from keychain
alias ssh='autossh -M0 -- -p $(getpw 1) /usr/local/bin/ssh'
```


autossh Version 1.4
-------------------

Building and Installing Autossh
--------------------------------

With version 1.4, autossh now uses autoconf. So the build procedure
is now the well-known:

	./configure
	make
	make install

Look at autossh.host for an example wrapper script.


Usage
-----
	autossh [-M <port>[:echo_port]] [-f] [SSH OPTIONS]

Description
-----------

autossh is a program to start a copy of ssh and monitor it, restarting
it as necessary should it die or stop passing traffic.

The original idea and the mechanism were from rstunnel (Reliable SSH
Tunnel). With version 1.2 the method changed: autossh now uses ssh to
construct a loop of ssh forwardings (one from local to remote, one
from remote to local), and then sends test data that it expects to get
back. (The idea is thanks to Terrence Martin.) 

With version 1.3, a new method is added (thanks to Ron Yorston): a
port may be specified for a remote echo service that will echo back
the test data. This avoids the congestion and the aggravation of
making sure all the port numbers on the remote machine do not
collide. The loop-of -forwardings method remains available for
situations where using an echo service may not be possible.

autossh has only three arguments of its own:

 -M <port>[:echo_port], to specify the base monitoring port to use, or
	alternatively, to specify the monitoring port and echo service
	port to use. 

	When no echo service port is specified, this port and the port 
	immediately above it (port# + 1) should be something nothing 
	else is using. autossh will send test data on the base monitoring 
	port, and receive it back on the port above. For example, if you 
	specify "-M 20000", autossh will set up forwards so that it can 
	send data on port 20000 and receive it back on 20001.

	Alternatively a port for a remote echo service may be
	specified. This should be port 7 if you wish to use the
	standard inetd echo service.  When an echo port is specified,
	only the specified monitor port is used, and it carries the
	monitor message in both directions.

	Many people disable the echo service, or even disable inetd,
	so check that this service is available on the remote
	machine. Some operating systems allow one to specify that the
	service only listen on the localhost (loopback interface),
	which would suffice for this use.

	The echo service may also be something more complicated:
	perhaps a daemon that monitors a group of ssh tunnels.

	-M 0 will turn the monitoring off, and autossh will only
	restart ssh on ssh exit.

	For example, if you are using a recent version of OpenSSH, you 
	may wish to explore using the ServerAliveInterval and 
	ServerAliveCountMax options to have the SSH client exit if it 
	finds itself no longer connected to the server. In many ways 
	this may be a better solution than the monitoring port.

 -f     Causes autossh to drop to the background before running ssh. The
        -f flag is stripped from arguments passed to ssh. Note that there
        is a crucial difference between the -f with autossh, and -f
        with ssh: when used with autossh, ssh will be *unable* to ask for
        passwords or passphrases. When -f is used, the "starting gate"
        time (see AUTOSSH_GATETIME) will be set to 0.

 -V     to have autossh display its version and exit.

All other arguments are passed to ssh. There are a number of
other settings, but these are all controlled through environment
variables. ssh seems to be appropriating more and more letters for
options, and this seems the easiest way to avoid collisions.

autossh tries to distinguish the manner of death of the ssh process it
is monitoring and act appropriately. The rules are:

   - If the ssh process exited normally (for example, someone typed
     "exit" in an interactive session), autossh exits rather than 
     restarting;
   - If autossh itself receives a SIGTERM, SIGINT, or a SIGKILL
     signal, it assumes that it was deliberately signalled, and exits
     after killing the child ssh process;
   - If autossh itself receives a SIGUSR1 signal, it will kill the child
     ssh process and start a new one;
   - Periodically (by default every 10 minutes), autossh attempts to pass
     traffic on the monitor forwarded port. If this fails, autossh will
     kill the child ssh process (if it is still running) and start a new
     one; 
   - If the child ssh process dies for any other reason, autossh will
     attempt to start a new one.

Startup behaviour:

   - If the ssh session fails with an exit status of 1 on the very first 
     try, autossh will assume that there is some problem with syntax or
     the connection setup, and will exit rather than retrying;
   - There is now a "starting gate" time. If the first ssh process fails 
     within the first few seconds of being started, autossh assumes that 
     it never made it "out of the starting gate", and exits. This is to handle
     initial failed authentication, connection, etc. This time is 30 seconds
     by default, and can be adjusted (see the AUTOSSH_GATETIME environment
     variable below).
   - NOTE: If AUTOSSH_GATETIME is set to 0, then BOTH of the above
           behaviours are disabled. This is useful for, for example,
	   having autossh start on boot. The "starting gate" time is
	   also set to 0 with the -f flag to autossh is used.

Continued failures:

   - If the ssh connection fails and attempts to restart it fail in
     quick succession, autossh will start delaying its attempts to
     restart, gradually backing farther and farther off up to a
     maximum interval of the autossh poll time (usually 10 minutes).
     autossh can be "prodded" to retry by signalling it, perhaps with
     SIGHUP ("kill -HUP").

Connection Setup
----------------

As connections must be established unattended, the use of autossh
requires that some form of automatic authentication be set up. The use
of RSAAuthentication with ssh-agent is the recommended method. The
example wrapper script attempts to check if there is an agent running
for the current environment, and to start one if there isn't.

It cannot be stressed enough that you must make sure ssh works on its
own, that you can set up the session you want before you try to
run it under autossh.

If you are tunnelling and using an older version of ssh that does not
support the -N flag, you should upgrade (your version has security
flaws). If you can't upgrade, you may wish to do as rstunnel does, and
give ssh a command to run, such as "sleep 99999999999".

Disabling connection monitoring
-------------------------------

A monitor port value of "0" ("autossh -M 0") will disable use of
the monitor ports; autossh will then only react to signals and the
death of the ssh process.

Environment Variables
---------------------

The following environment variables can be set:

    AUTOSSH_DEBUG	  - sets logging level to LOG_DEBUG, and if
			    the operating system supports it, sets
			    syslog to duplicate log entries to stderr.
    AUTOSSH_FIRST_POLL	  - time to initial poll (default is as 
			    AUTOSSH_POLL below).
    AUTOSSH_GATETIME      - how long ssh must be up before we consider
	                    it a successful connection. Default is 30
			    seconds. If set to 0, then this behaviour
			    is disabled, and as well, autossh will retry
			    even on failure of first attempt to run ssh.
    AUTOSSH_LOGFILE	  - sets autossh to use the named log file,
			    rather than syslog.
    AUTOSSH_LOGLEVEL	  - log level, they correspond to the levels 
			    used by syslog; so 0-7 with 7 being the
			    chattiest.
    AUTOSSH_MAXLIFETIME   - Sets the maximum number of seconds the process 
			    should live for before killing off the ssh child 
			    and exiting.
    AUTOSSH_MAXSTART	  - specifies how many times ssh should be started.
			    A negative number means no limit on the number 
			    of times ssh is started. The default value is -1.
    AUTOSSH_MESSAGE	  - append a custom message to the echo string (max 64
			    bytes).
    AUTOSSH_NTSERVICE     - when set to "yes" , setup autossh to run as an 
			    NT service under cygrunsrv. This adds the -N flag
			    for ssh if not already set, sets the log output 
			    to stdout, and changes the behaviour on ssh exit 
			    so that it will restart even on a normal exit.
    AUTOSSH_PATH	  - path to the ssh executable, in case
			    it is different than that compiled in.
    AUTOSSH_PIDFILE	  - write autossh pid to specified file.
    AUTOSSH_POLL	  - poll time in seconds; default is 600.
    			    Changing this will also change the first
			    poll time, unless AUTOSSH_FIRST_POLL is
			    used to set it to something different.
			    If the poll time is less than twice the 
			    network timeouts (default 15 seconds) the 
			    network timeouts will be adjusted downward 
			    to 1/2 the poll time.
    AUTOSSH_PORT	  - set monitor port. Mostly in case ssh
			    appropriates -M at some time. But because
			    of this possible use, AUTOSSH_PORT overrides
			    the -M flag.

SSH Options
------------------

There are two particular OpenSSH options that are useful when using
autossh:

1) ExitOnForwardFailure=yes on the client side to make sure forwardings
have succeeded when autossh assumes the connection is setup properly.

2) ClientAliveInterval on the server side to make sure the listening
socket is closed on the server side if the connection closes on the
client side.

Logging and Syslog
------------------

autossh logs to syslog using the LOG_USER facility. Your syslog may
have to be configured to accept messages for this facility. This is
usually done in /etc/syslog.conf.

-- 
Kudos and raspberries to harding [at] motd.ca
