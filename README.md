# RPi-XMPP-Server

This repository holds essential files to get an XMPP server running on a Raspberry Pi

## Disclaimer

RPi-XMPP-Server is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version. RPi-XMPP-Server is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with RPi-XMPP-Server.  If not, see <http://www.gnu.org/licenses/>.

This project contains work from other open-source project, or other free project. This includes Prosody (IM client), and Pi-Shrink (a script to shrink RPi image). Other projects, including Raspberry Pi OS and all the softwares contained in the image or in it's (apt) source, are also presented in the released image. Any credit about them should goes to their corrisponding authors or maintainers.

## Applicable scenarios

XMPP is a relatively old IM (Instant Messaging), so it’s not a good idea to use it in most cases. Good replacements, like Telegram, Line, WhatsApp, or even the built in iMessage of iPhone, can easily replace most of the functions of XMPP. Only under a handful of scenarios is this project useful, for example:

- When there’s no access to internet, or certain services on the internet. Users may be under a LAN, and require a local messaging method.
- When the user/admin would like to have full control over the network/servers involved in the application. Sometimes when the conversation is private or confidential, the users may wish to avoid using public servers or to avoid transmitting on public network. XMPP is relatively easy to setup, and most importantly, has a fully developed solution on almost every platform for clients.
- When custom encryption is required. XMPP transmits only text, and it’s possible to encrypt the text before sending, implementing a custom encryption system.

## Requirements

To use this project, following items are required:

- A Raspberry Pi. It will serve as a server, so preferably models with higher processing ability and good connection speed. A RPi 4B 4G version will do nicely, and only costs about 50£.
- A MicroSD card. RPi doesn’t come with persistent storage for file systems, so a MicroSD card is required. Minimum runnable space is 4GB, though it is recommended to use a 32GB or above MicroSD card for future maintenance.
- Internet connection. This is required for the downloading and configuration of this project.
- A computer with any image flashing tool, and SSH client. Image flashing tool can be Win32 Imager, BalenaEtcher, Raspberry Pi Imager, or any other similar tools.

## Usage

### Using the image provided

This is the easiest and the recommended way to use this project.

#### Login

To use the image provided, first download the image, and flash it into a prepared MicroSD card. Plug it into RPi and power it up. By default **SSH is enabled with username and password**. The default user is ``rpixmpp``, default password is ``password``. **Change this immediately after logging into the system with:**

```shell
$ passwd
Changing password for rpixmpp:
New password:
```

Enter and confirm the new password. Since the server is normally maintained by one or few admin, it’s recommended to disable password login and use key login instead. To use key login, first use:

```shell
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/rpixmpp/.ssh/id_rsa): <directly press enter or enter your own directory>
Created directory '/home/rpixmpp/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/rpixmpp/.ssh/id_rsa.
Your public key has been saved in /home/rpixmpp/.ssh/id_rsa.pub.
The key fingerprint is:
```

To generate key pair. Keep in mind that private key is **NOT** to be transmitted over internet or any unsecured means, or the security would be compromised. Use local copy instead to move the private key into your client. Or better, generate the key pair on your client, and send the public key to the server. This can also be done using some advanced SSH client. After getting the private key into your client, remove the server copy.

Then configure the server to use the key as:

```shell
$ cd .ssh
$ cat id_rsa.pub >> authorized_keys
```

And disable password login by editing the corresponding lines in config file ``/etc/ssh/sshd_config`` into:

```lua
RSAAuthentication yes
PubkeyAuthentication yes
PermitRootLogin yes
PasswordAuthentication no
```

Then restart SSH with:

```shell
$ sudo systemctl restart sshd
```

Keep in mind that this means once anything happened to your client, eg. reinstalling system, losing local private key copy; you wouldn’t be able to SSH into the machine. The only way is to directly access the server, and reconfigure it. **SO DO NOT DISABLE LOCAL LOGIN!**

#### Configuring prosody

This project is using [Prosody](https://prosody.im), a free modern XMPP communication server, to achieve IM function. The prosody is already installed, and preconfigured to an immediately useable state. However for security and better experience, it’s recommended to configure it. All of the preference of prosody is configured via a file at ``/etc/prosody/prosody.cfg.lua``. The following configuration is not all necessary, please read carefully before proceeding.

- If you want to use another host name or use actual domain name (not recommended, if you’ve got a domain name, why not get an actual server?), change ``VirtualHost`` into something you want. Note that if this is changed, all of the account relevant settings need to be changed too.
- If you want multiple admins, add them into ``admins`` and separate with commas.
- The default admin account is ``admin@xmpprpi``, default password is ``password``. Change immediately with ``sudo prosodyctl passwd admin@xmpprpi``.
- If you wish to enable or disable additional functions like offline messaging, message archiving, etc., please refer to official documents.

Now, the server can be used. Client registration is enabled, so users only need to use their favorite XMPP client, and configure the server IP (this is necessary as the host name is not resolvable), then follow the client’s guide to register and use. Under cases where there’s no direct route to the server, eg. the server is behind NAT, ZeroTier is installed to provide a virtual LAN.

#### Configure ZeroTier

To use ZT, an account is needed for configuration. Only network manager needs an account, normal users **don’t need one, they only need a ZT client**. Follow the instruction on the [ZT website](https://www.zerotier.com) to register, free uses can create networks up to 500 users can join. If only going through NAT is required, no further config on ZT is needed. The user only need to put the network ID into their clients and await admin to approve their joining request.

The RPi server needs to join ZT network too. But before that, a new ZT ID should be generated, because this image is openly distributed. Use the following command to generate new ID:

```shell
$ sudo rm /var/lib/zerotier-one/identity.public
$ sudo rm /var/lib/zerotier-one/identity.secret
$ sudo systemctl stop zerotier-one.service
$ sudo systemctl start zerotier-one.service
```

Then use CLI to join the network:

```shell
$ zerotier-cli join <network ID>
```

Don’t forget to approve the join request on the web console. This will make all of the ports reachable from ZT interface. For security reasons, it’s necessary to limit this. This can be done by configuring the firewall on RPi to only accept traffic from ZT destined to certain ports, or, if it’s required for the admin to manage the server through ZT, can also be done on ZT web console by setting up flow rules. For detailed guide please refer to either official documents or someone with experience, as messing around with firewalls can easily make your device unreachable, or even worse.

#### Group chat

XMPP supports “Chat Rooms”. The implementation in prosody is ``mod_disco``, which implements XEP-0030. The admin can configure multiple different chat rooms, and if configured correctly, can be discovered by clients. To use this function, first enable this mod in ``Other modules``, like (replace examples with your config):

```lua
    modules_enabled = {
        -- Other modules
        "disco"; -- Enable mod_disco
    }
    VirtualHost "example.com"
        disco_items = {
            { "channels.example.net", "Public channels" };
            { "irc.jabberfr.org", "A public IRC gateway using biboumi" };
        }
 
    Component "secret.example.com"
        -- Prevent a service discovery on example.com from showing this component:
        disco_hidden = true

```

Since we’re not using actual domains, these rooms won’t be subdomains, thus need to be configured manually in the ``disco_items`` section.

#### File sharing

As built-in function, XMPP supports synchronous file sharing methods like SI File Transfer (XEP-0096), or, with the support of MAM, also asynchronous methods like SIMS (XEP-0385). This makes the single client-to-client file transfer easy, but has several downsides. Asynchronous method puts tolls on the server storage space, and group sharing files, since they go through the server, puts tolls on server band width. A better option is to use the server as a file drop server.

#### Voice/video call

XMPP does not natively handles the video and voice call, but can be extended to do that. Most of the methods require a dedicated client, and the XMPP channel is only used to initiate the handshake, while the actual calling traffic is P2P. This normally requires good internet connection, and a way to proxy the traffic between NAT. Theoretically, with ZT running, clients can directly call each other without extra step. However, multi-attendance call is not supported by most clients, and is recommended to accomplish with other dedicated software. As group conference call requires large amount of bandwidth, it’s not recommended to host such systems on RPi in home environment.

### Build with script provided

TBD. A universal script is a huge workload, help is welcomed.

### Manual build

TBD.
