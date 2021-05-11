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

- Separate your VPN credentials from your NetVM and AppVMs.
- Concretely prevent network traffic from leaking _around_ the VPN.
- Easily control which of your AppVMs are connected through your VPN by simply setting it as a NetVM of the desired AppVM.




Set up a ProxyVM as a VPN gateway using the *qubes-tunnel* service
----------------------------------------------------------------

This method has extended anti-leak features that also make the connection _fail closed_ should it be interrupted[1]. It also has the advantage of using configuration files tailored by your VPN service provider so it may be the easiest way to setup a working and secure link.

The following has been tested with Fedora 33 and Debian 10 templates:

1. Create a new VPN VM

    Launch the qube/VM creation tool, give the qube a name like _sys-vpn_, select a Debian or Fedora Template, and choose _sys-net_ or _sys-firewall_ for Networking. Next, select _'provides network'_ then click _OK_.

   ![Create\_New\_VM.png](/attachment/wiki/VPN/Create_New_VM.png)

   Note:  If your choice of TemplateVM doesn't already have the VPN client software, you'll need to install its package such as `openvpn` in the template before proceeding; qubes-tunnel uses openvpn by default. If your TemplateVM is one of the "minimal" types, you may need to install additional networking packages as well .....PUT LINK HERE.....

2. Additional VM settings

   A settings window for the new VPN VM should appear from the last step. Choose the _Services_ tab, then choose 'custom' in the dropdown box and click _Add_. then add 'qubes-tunnel'. Finally, for the service name enter 'qubes-tunnel' and click _OK_.

3. Set up the VPN client.

   Make sure the new VPN VM and its TemplateVM are not running.
   Run a terminal (CLI) in the VPN VM -- this will start the VM.
   Then create a new `/rw/config/qtunnel` folder with:

       sudo mkdir /rw/config/qtunnel


    Obtain the configuration files from your VPN service provider[2] then copy them to the `/rw/config/qtunnel` folder. Finally, copy the desired config file to `qtunnel.conf`. For example:

   ~~~
   $ cd /rw/config/qtunnel
   $ sudo unzip ~/acme-ovpn-example.zip
   Archive:  acme-ovpn-example.zip
    extracting: US_Northeast.conf
    extracting: US_Northwest.conf
    extracting: US_Southeast.conf
    extracting: US_Southwest.conf
    extracting: acme_vpn_ca.crt
   
   $ sudo cp US_Northeast.conf qtunnel.conf
   ~~~


4. Test your client configuration!

   Run the VPN client as root from a CLI prompt in the 'qtunnel' folder.

   For example:

       sudo openvpn --config qtunnel.conf --auth-user-pass

   Watch for status messages that indicate whether the connection is successful; In the case of `openvpn` it should display _"Initialization sequence completed"_. Testing can be done from another VPN VM terminal window using a `ping` command like:

       ping -c20 1.1.1.1

   Optional – DNS may be tested at this point by replacing addresses in `/etc/resolv.conf` with ones appropriate for your VPN (although this file will not be used when setup is complete). Refer to template documentation for details.

   Diagnose any connection problems at this stage by using resources such as client documentation and help from your VPN service provider.

   Terminate `openvpn` by pressing _ctrl-c_, then proceed to the next step when you're sure the basic VPN connection works.

5. Download and install qubes-tunnel

   At a CLI prompt in the VPN VM, clone the qubes-tunnel repository:

      $ git clone https://github.com/QubesOS-contrib/qubes-tunnel.git

   (Optionally, signatures [can be verified](https://git-scm.com/docs/git-verify-commit) at this point for security.)

   Next, type the following to automatically install qubes-tunnel, enable firewall rules and save your login information:

      $ cd qubes-tunnel
      sudo /usr/lib/qubes/qtunnel-setup --config

   If username and password aren't used by your VPN provider you may leave these blank.

   On the _Services_ tab of the VPN VM settings, add `qubes-tunnel` and make sure it is checked, then click 'OK'. This starts the qubes-tunnel service when the proxyVM starts.

6. Restart the new VM!

The link should then be established automatically with a popup notification to that effect.


Usage
-----

- Configure your AppVMs to use the VPN VM as a NetVM...

![Settings-NetVM.png](/attachment/wiki/VPN/Settings-NetVM.png)

- If you wish to use the [Qubes firewall](/doc/firewall), the settings can be added directly to the AppVM; The VPN VM functions as a firewall VM.

- FIXME FOR R4.0/4.1....... If you want to update your TemplateVMs through the VPN, enable the `qubes-updates-proxy` service in your new FirewallVM.
You can do this in the Services tab in Qubes VM Manager or on the command-line:

    qvm-service -e <name> qubes-updates-proxy


Troubleshooting
---------------

* The `systemctl` command can be used to control the VPN client, which is running as a systemd service called `qubes-tunnel.service`.
* To view the log when qubes-tunnel is running, use `journalctl -u qubes-tunnel`.
* Test DNS: Ping a familiar domain name from an AppVM. It should print the IP address for the domain.
* Use `iptables -L -v` and `iptables -L -v -t nat` to check firewall rules. The latter shows the critical PR-QBS chain that enables DNS forwarding.

Notes
-----

[1] This firewall and DNS configuration prevent any packets from being forwarded to the 'bare' upstream interface (eth0).

[2] Most VPN providers with OpenVPN access offer tailored configuration files for the user to download and use. These are typically avaiable in the "Support" section of the providers' website.

[3] Routing of traffic through the tunnel is usually preconfigured by the VPN provider; This happens automatically and can be seen in `openvpn` log output as `redirect-gateway def1`. If necessary, the user can manually add this directive to the ovpn/conf file.
