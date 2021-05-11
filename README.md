---
layout: doc
title: VPN
permalink: /doc/vpn/
redirect_from:
- /doc/privacy/vpn/
- /en/doc/vpn/
- /doc/VPN/
- /wiki/VPN/
---

How To make a VPN Tunnel in Qubes
==================================

Although setting up a VPN connection is not by itself Qubes specific, Qubes includes a number of tools that can make the client-side setup of your VPN more versatile and secure. This document is a Qubes-specific outline for choosing the type of VM to use, and shows two methods for preparing the preferred VM type (ProxyVM). The first method uses NetworkManager, while the second method if the most fail-safe and demonstrates the new qubes-tunnel feature.

When considering the specific configuration options for your connection you may also need to refer to your template OS and VPN service provider's documentation. However, if you are comfortable with basic CLI
commands and you have downloaded the configuration files from your VPN provider, then the instructions
below should be adequate to setup a working VPN connection.

### Proxy VMs That Provide Network Services

One of the best unique features of Qubes OS is its special type of qube that we'll call a ProxyVM in this document. The special thing is that your AppVMs see this as a NetVM (or uplink), and your NetVMs (sys-net) see it as a downstream. Because of this, you can place a ProxyVM between your AppVMs and your NetVM where it functions much like a router. This is how the default sys-firewall qube functions.

Using a ProxyVM to set up a VPN client gives you the ability to:

- Separate your VPN credentials from your NetVM.
- Separate your VPN credentials from your AppVM data.
- Easily control which of your AppVMs are connected to your VPN by simply setting it as a NetVM of the desired AppVM.



Set up a ProxyVM as a VPN gateway using the *qubes-tunnel* service
----------------------------------------------------------------

This method has extended anti-leak features that also make the connection _fail closed_ should it be interrupted[1]. It also has the advantage of using configuration files tailored by your VPN service provider so it may be the easiest way to setup a working and secure link.

The following has been tested with Fedora 33 and Debian 10 templates:

1. Create a new VM, name it, click the ProxyVM radio button, and choose a color and template.

   ![Create\_New\_VM.png](/attachment/wiki/VPN/Create_New_VM.png)

   Note: Do not enable NetworkManager in the ProxyVM, as it can interfere with the scripts' DNS features. If you enabled NetworkManager or used other VPN setup methods in a previous attempt, do not re-use the old ProxyVM... Create a new one according to this step.

   If your choice of TemplateVM doesn't already have the VPN client software, you'll need to install its package such as `openvpn` in the template before proceeding. If your TemplateVM is one of the "minimal" types, you may need to install additional networking packages as well .....PUT LINK HERE.....

2. Set up the VPN client.

   Make sure the new VPN VM and its TemplateVM are not running.
   Run a terminal (CLI) in the VPN VM -- this will start the VM.
   Then create a new `/rw/config/qtunnel` folder with:

       sudo mkdir /rw/config/qtunnel

   
    Obtain the configuration files from your VPN service provider[2] then copy them to the `/rw/config/qtunnel` folder. Finally, copy or link the desired config file to `qtunnel.conf`. For example:

   ~~~
   cd /rw/config/qtunnel
   sudo unzip ~/ovpn-configs-example.zip
   sudo ln -s US_East.ovpn qtunnel.conf
   ~~~

   Notes on configuration contents:

      * Files accompanying the main config such as `*.crt` and `*.pem` should also go to `/rw/config/qtunnel` folder.
      * Files referenced in `qtunnel.conf` should not use absolute paths such as `/etc/...`.
      * Typical VPN configs will specify a `tun` interface; note that `tap` interfaces are untested with qubes-tunnel.
      * The configuration should also route all traffic through your VPN's tunnel interface once a connection is started. See footnotes[3].

3. Test your client configuration!

   Run the VPN client from a CLI prompt in the 'qtunnel' folder, preferably as root.

   For example:

       sudo openvpn --cd /rw/config/qtunnel --config qtunnel.conf --verb 5

   Watch for status messages that indicate whether the connection is successful and test from another VPN VM terminal window with `ping`.

       ping -c20 1.1.1.1

   Optional – DNS may be tested at this point by replacing addresses in `/etc/resolv.conf` with ones appropriate for your VPN (although this file will not be used when setup is complete). Refer to template documentation for details.

   Diagnose any connection problems at this stage by using resources such as client documentation and help from your VPN service provider.

   Proceed to the next step when you're sure the basic VPN connection is working.

4. Download and install qubes-tunnel

   At a CLI prompt in the VPN VM, clone the qubes-tunnel repository and verify signature(s):

      $ git clone https://github.com/QubesOS-contrib/qubes-tunnel.git
      $ cd qubes-tunnel
      $ git log --show-signature -1

   Next, type the following to automatically install qubes-tunnel, enable firewall rules and save your login information:

      sudo /usr/lib/qubes/qtunnel-setup --config

   If username and password aren't used by your VPN provider you may leave these blank.

   On the _Services_ tab of the VPN VM settings, add `qubes-tunnel` and make sure it is checked, then click 'OK'. This starts the qubes-tunnel service when the proxyVM starts.

5. Restart the new VM!

The link should then be established automatically with a popup notification to that effect.


Usage
-----

- Configure your AppVMs to use the VPN VM as a NetVM...

![Settings-NetVM.png](/attachment/wiki/VPN/Settings-NetVM.png)

- If you wish to use the [Qubes firewall](/doc/firewall), the settings can be added directly to the AppVM; The VPN VM functions as a firewall VM.

- FIXME FOR R4.0....... If you want to update your TemplateVMs through the VPN, enable the `qubes-updates-proxy` service in your new FirewallVM.
You can do this in the Services tab in Qubes VM Manager or on the command-line:

    qvm-service -e <name> qubes-updates-proxy

Then, configure your templates to use your new FirewallVM as their NetVM.

- The `systemctl` command can be used to control the VPN client, which is running as a systemd service called `qubes-tunnel.service`.

Troubleshooting
---------------

* To view the log, use `journalctl -u NetworkManager` or `journalctl -u qubes-tunnel` as needed.
* Test DNS: Ping a familiar domain name from an appVM. It should print the IP address for the domain.
* Use `iptables -L -v` and `iptables -L -v -t nat` to check firewall rules. The latter shows the critical PR-QBS chain that enables DNS forwarding.

Notes
-----

[1] This firewall and DNS configuration prevent any packets from being forwarded to the 'bare' upstream interface (eth0). It will also isolate the VPN VM from inadventant use of the tunnel, but this extra protection is not available for the Network Manager configuration.

[2] Most VPN providers with OpenVPN access offer tailored configuration files for the user to download and use. FIXME: maybe show download links for popular/respected VPN services.

[3] Routing of traffic through the tunnel is usually preconfigured by the VPN provider; This happens automatically and can be seen in the log output as `redirect-gateway def1`. If necessary, the user can manually add this directive to the ovpn/conf file.
