OpenConnect VM
==============

Simple Vagrant configuration for a local OpenConnect-ready VM that can serve as a web traffic proxy.

Configuration
-------------

Create a `config.yml` file in the `/vars` directory with your server address, username, and password. Use `/vars/config.yml.sample` as an example for how this should be set up.

Next, run `vagrant up` to build and configure the VM.

SOCKS Proxy
-----------

The server will be automatically set up to act as a SOCKS5 proxy - you can route all web, SSH, and other traffic through the server to the configured VPN.

The proxy is listening on port `1080`.

Under VirtualBox, the server will automatically start with a local IP of `192.168.90.10` (use this when configuring proxied applications).

Under Hyper-V, you will need to identify the dynamically-provisioned IP address after boot, but can likely use the `openconnect` host alias during configuration.

### Browser Configuration

Various browsers work a bit differently with SOCKS proxies.

#### Firefox

Firefox is relatively straight-forward. In the Advanced section of the settings screen, you can configure network settings. Add a SOCKS5 proxy pointing to `openconnect` on port `1080`. Don't forget to add `openconnect` itself to the "no proxy" list or Firefox will try to grab the DNS entries for your proxy server from the proxy itself and end up in an infinite loop!

![Firefox configuration](https://s3.amazonaws.com/uploads.hipchat.com/52421/365110/YbBNENfUGjXl60L/upload.png)

#### Safari

Thankfully, Safari's proxy configuration is somewhat similar to Firefox's. Just set the proxy settings in the network configuration.

#### Chrome

Chrome can be a bit tricky; particularly on Windows where it uses _system-level_ proxy configuration by default. Instead of editing network configuration within the application, you must launch Chrome with a set of command-line flags that will then start it up using the proxy instead of the defaults.

On Mac OS you can add the following script to your `~/.bash_profile` to launch or relaunch Chrome automatically:

```sh
proxyChrome() {
	local proxy=("open" "-a" "\"Google Chrome\"" "--args");
	if [ "$1" = "proxyon" ]
		then
		proxy+=("--proxy-server=\"socks5://openconnect\"")
		proxy+=("--proxy-bypass-list=\"openconnect;*.google.com;*twitter.com;*facebook.com;localhost;127.0.0.1\"")
	elif [ "$1" = "proxyoff" ]
		then
		proxy+=("--no-proxy-server")
	fi

	if [ ! -z "{$proxy[@]}" ]
		then
		if pgrep "Google Chrome" > /dev/null
			then
			killall "Google Chrome"
			proxy+=("--restore-last-session")
			sleep 1
		fi
		eval "${proxy[@]}"
	fi
}
```

A similar script could likely be achieved on any bash-style system, but the `open` command is only available in Mac OS.

### Shell Configuration

If you need to use Git or other shell-based tools over the SOCKS proxy, you can configure SSH to use SOCKS through its 
 configuration file (`~/.ssh/config`):

```sh
Host {{ VPN-protected server }}
	ProxyCommand=nc -X 5 -x openconnect:1080 %h %p
```

### Utility

For convenience, those of you using a Bash-type shell can add a script to your bash profile for remotely turning the SOCKS proxy on and off from anywhere on your system:

```sh
cntrlVPN() {
	local current=$PWD
	cd {{ location where you cloned openconnect-vm }}
	vagrant $1
	cd $current
}
alias vpn=cntrlVPN
```

Now, you can type `vpn up` and `vpn halt` from any location in a terminal to activate/deactivate the SOCKS proxy at will.

If you're also using the Chrome proxy script above, you can add the following between the `vagrant $1` and `cd $current` lines to automatically restart Chrome with or without proxy support when you bring the proxy up and down:

```sh
if [ "$1" = "up" ] || [ "$1" = "reload" ]
	then
	proxyChrome proxyon
elif [ "$1" = "halt" ] || [ "$1" = "suspend" ] || [ "$1" = "destroy" ]
	then
	proxyChrome proxyoff
fi
```

Frequently Asked Questions
--------------------------

### My password was changed for the VPN, how do I let the VM know?

Just update the values in `/vars/config.yml` and re-run `vagrant up`. This will both update machine and reset any configuration values necessary.

Credits
-------

The Ansible provisioning [was originally built by Crown Copyright (Government Digital Service)](https://github.com/alphagov/ansible-role-openconnect)

Special thanks to [Luke Woodward](https://github.com/lkwdwrd) for testing support and Bash alias/functions!
