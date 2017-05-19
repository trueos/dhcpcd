dhcpcd - DHCP client daemon
Copyright (c) 2006-2016 Roy Marples <roy@marples.name>

**DHCPCD Information: Table of Contents**

- [Installation](#installation)
- [Notes](#notes)
- [Hooks](#hooks)
- [Compatibility](#compatibility)
- [Upgrading](#upgrading)
- [Changelog](#changelog)

**General Project Information: Table of Contents**

- [General TrueOS Information](#gentrosinfo)
	- [TrueOS Project Documentation](#docs)
		- [TrueOS Handbook](#trueosdoc)
		- [Lumina Handbook](#luminadoc)
		- [SysAdm Handbooks](#sysadmdoc)
	- [Filing Issues or Feature Requests](#fileissues)
	- [Community Channels](#community)
		- [Discourse](#discourse)
		- [Gitter](#gitter)
		- [IRC](#irc)
		- [Subreddit](#subreddit)
	- [Social Media](#socialmedia)

<!-- END GENERAL INFO TOC -->

Installation
------------
./configure; make; make install
man dhcpcd for command line options
man dhcpcd.conf for configuration options
man dhcpcd-run-hooks to learn how to hook scripts into dhcpcd events


Notes
-----
To compile small dhcpcd, maybe to be used for installation media where
size is a concern, you can use the --small configure option to enable
a reduced feature set within dhcpcd.
Currently this just removes non important options out of
dhcpcd-definitions.conf, the custom logger and
support for DHCPv6 Prefix Delegation.
Other features maybe dropped as and when required.
dhcpcd can also be made smaller by removing the IPv4 or IPv6 stack:
  *  --disable-inet
  *  --disable-inet6
Or by removing the following features:
  *  --disable-auth
  *  --disable-arp
  *  --disable-arping
  *  --disable-ipv4ll
  *  --disable-dhcp6

You can also move the embedded extended configuration from the dhcpcd binary
to an external file (LIBEXECDIR/dhcpcd-definitions.conf)
	--disable-embedded
If dhcpcd cannot load this file at runtime, dhcpcd will work but will not be
able to decode any DHCP/DHCPv6 options that are not defined by the user
in /etc/dhcpcd.conf. This does not really change the total on disk size.

If you're cross compiling you may need set the platform if OS is different
from the host.
--target=sparc-sun-netbsd5.0

If you're building for an MMU-less system where fork() does not work, you
should ./configure --disable-fork.
This also puts the --no-background flag on and stops the --background flag
from working.

You can change the default dirs with these knobs.
For example, to satisfy FHS compliance you would do this:-
./configure --libexecdir=/lib/dhcpcd dbdir=/var/lib/dhcpcd

We now default to using -std=c99. For 64-bit linux, this always works, but
for 32-bit linux it requires either gnu99 or a patch to asm/types.h.
Most distros patch linux headers so this should work fine.
linux-2.6.24 finally ships with a working 32-bit header.
If your linux headers are older, or your distro hasn't patched them you can
set CSTD=gnu99 to work around this.

Some BSD systems do not allow the manipulation of automatically added subnet
routes. You can find discussion here:
    http://mail-index.netbsd.org/tech-net/2008/12/03/msg000896.html
BSD systems where this has been fixed or is known to work are:
    NetBSD-5.0
    FreeBSD-10.0

Some BSD systems protect against IPv6 NS/NA messages by ensuring that the
source address matches a prefix on the recieved by a RA message.
This is an error as the correct check is for on-link prefixes as the
kernel may not be handling RA itself.
BSD systems where this has been fixed or is known to work are:
    NetBSD-7.0
    OpenBSD-5.0
    patch submitted against FreeBSD-10.0

Some BSD systems do not announce IPv6 address flag changes, such as
IN6_IFF_TENTATIVE, IN6_IFF_DUPLICATED, etc. On these systems,
dhcpcd will poll a freshly added address until either IN6_IFF_TENTATIVE is
cleared or IN6_IFF_DUPLICATED is set and take action accordingly.
BSD systems where this has been fixed or is known to work are:
    NetBSD-7.0

OpenBSD will always add it's own link-local address if no link-local address
exists, because it doesn't check if the address we are adding is a link-local
address or not.

Some BSD systems do not announce cached neighbour route changes based
on reachability to userland. For such systems, IPv6 routers will always
be assumed to be reachable until they either stop being a router or expire.
BSD systems where this has been fixed or is known to work are:
    NetBSD-7.99.3

Linux prior to 3.17 won't allow userland to manage IPv6 temporary addresses.
Either upgrade or don't allow dhcpcd to manage the RA,
so don't set either "ipv6ra_own" or "slaac private" in dhcpcd.conf if you
want to have working IPv6 temporary addresses.
SLAAC private addresses are just as private, just stable.

ArchLinux presently sanitises all kernel headers to the latest version
regardless of the version for your CPU. As such, Arch presently ships a
3.12 kernel with 3.17 headers which claim that it supports temporary address
management and no automatic prefix route generation, both of which are
obviously false. You will have to patch support either in the kernel or
out of the headers (or dhcpcd itself) to have correct operation.

We try and detect how dhcpcd should interact with system services at runtime.
If we cannot auto-detect how do to this, or it is wrong then
you can change this by passing shell commands to --serviceexists,
--servicecmd and optionally --servicestatus to ./configure or overriding
the service variables in a hook.

Some systems have /dev management systems and some of these like to rename
interfaces. As this system would listen in the same way as dhcpcd to new
interface arrivals, dhcpcd needs to listen to the /dev management sytem
instead of the kernel. However, if the /dev management system breaks, stops
working, or changes to a new one, dhcpcd should still try and continue to work.
To facilitate this, dhcpcd allows a plugin to load to instruct dhcpcd when it
can use an interface. As of the time of writing only udev support is included.
You can disable this with --without-dev, or without-udev.
NOTE: in Gentoo at least, sys-fs/udev as provided by systemd leaks memory
sys-fs/eudev, the fork of udev does not and as such is recommended.

dhcpcd uses eloop.c, which is a portable main event loop with timeouts and
signal handling. Unlike libevent and similar, it can be transplanted directly
within the application - the only caveat outside of POSIX calls is that
you provide queue.h based on a recent BSD (glibc sys/queue.h is not enough).
eloop supports the following polling mechanisms, listed in order of preference:
	kqueue, epoll, pollts, ppoll and pselect.
If signal handling is disabled (ie in RTEMS or other single process
OS's) then eloop can use poll.
You can decide which polling mechanism dhcpcd will select in eloop like so
./configure --with-poll=[kqueue|epoll|pselect|pollts|ppoll]

To prepare dhcpcd for import into a platform source tree (like NetBSD)
you can use the make import target to create /tmp/dhcpcd-$version and
populate it with all the source files and hooks needed.
In this instance, you may wish to disable some configured tests when
the binary has to run on older versions which lack support, such as getline.
./configure --without-getline

Building for distribution (ie making a dhcpcd source tarball) now requires
gmake-4 or any BSD make.


Hooks
-----
Not all the hooks in dhcpcd-hooks are installed by default.
By default we install 01-test, 02-dump, 10-mtu, 20-resolv.conf and 30-hostname.
The other hooks, 10-wpa_supplicant, 15-timezone, 29-lookup-hostname
are installed to $(datadir)/dhcpcd/hooks by default and need to be
copied to $(libexecdir)/dhcpcd-hooks for use.
The configure program attempts to find hooks for systems you have installed.
To add more simply
./configure -with-hook=ntp.conf

Some system services expose the name of the service we are in,
by default dhcpcd will pick RC_SVCNAME from the environment.
You can override this in CPPFLAGS+= -DRC_SVCNAME="YOUR_SVCNAME".
This is important because dhcpcd will scrub the environment aside from $PATH
before running hooks.
This variable could be used to facilitate service re-entry so this chain could
happen in a custom OS hook:
  dhcpcd service marked inactive && dhcpcd service starts
  dependant services are not started because dhcpcd is inactive (not stopped)
  dhcpcd hook tests if $if_up = true and $af_waiting is empty or unset.
  if true, mark the dhcpcd service as started and then start dependencies
  if false and the dhcpcd service was previously started, mark as inactive and
     stop any dependant services.


Compatibility
-------------
dhcpcd-5 is only fully command line compatible with dhcpcd-4
For compatibility with older versions, use dhcpcd-4


Upgrading
---------
dhcpcd-7 defaults the database directory to /var/db/dhcpcd instead of /var/db
and now stores dhcpcd.duid and dhcpcd.secret in there instead of /etc.
of /etc.
The Makefile _confinstall target will attempt to move the files correctly from
the old locations to the new locations.
Of course this won't work if dhcpcd-7 is packaged up, so packagers will need to
install similar logic into their dhcpcd package.


ChangeLog
---------
We no longer supply a ChangeLog.
However, you're more than welcome to read the commit log at
http://roy.marples.name/projects/dhcpcd/timeline/

# General TrueOS Information <a name="gentrosinfo"></a>

This section describes where you can find more information about TrueOS and its related projects, file new issues on GitHub, and converse with other users or contributors to the project.

## TrueOS Project Documentation <a name="docs"></a>

A number of [Sphinx](http://www.sphinx-doc.org/en/stable/) generated reStructuredText handbooks are available to introduce you to the TrueOS, Lumina, and SysAdm projects. These handbooks are open source, and users are always encouraged to open GitHub issues or fix any errors they find in the documentation.

### TrueOS Handbook <a name="trueosdoc"></a>

The [TrueOS User Guide](https://www.trueos.org/handbook/trueos.html) is a comprehensive guide to install TrueOS, along with post-installation configuration help, recommendations for useful utilities and applications, and a help and support section containing solutions for common issues and links to community and development chat channels for uncommon issues. There is also a chapter describing the experimental TrueOS Pico project and links to the Lumina and SysAdm documentation. All TrueOS documentation is hosted on the [TrueOS website](https://www.trueos.org).

### Lumina Handbook <a name="luminadoc"></a>

The Lumina Desktop Environment has its own [handbook](https://lumina-desktop.org/handbook/), hosted on the [Lumina Website](https://lumina-desktop.org). This handbook contains brief installation instructions. However, due to the highly customizable nature of Lumina, the focus of the handbook lies mainly in documenting all user configurable settings. Each option is typically described in detail, with both text and screenshots. Finally, the suite of unique Qt5 utilities included with Lumina are also documented.

TrueOS users are encouraged to review the Lumina documentation, as the Lumina Desktop Environment is installed by default with TrueOS.

### SysAdm Handbooks <a name="sysadmdoc"></a>

Due to complexity of this project, SysAdm documentation is split into three different guides:

1. **API Reference Guide** (https://api.sysadm.us/getstarted.html)

The Application Programming Interface (API) Reference Guide is a comprehensive library of all API calls and WebSocket requests for SysAdm. In addition to documenting all SysAdm subsystems and classes, the guide provides detailed examples of requests and responses, authentication, and SSL certificate management. This guide is constantly updated, ensuring it provides accurate information at all times.

2. **Client Handbook** (https://sysadm.us/handbook/client/)

The SysAdm Client handbook documents all aspects of the SysAdm client, as well as describing of the PC-BSD system utilities is replaces. Detailed descriptions of utilities such as Appcafe, Life Preserver, and the Boot Environment Manager are contained here, as well as a general guide to using these utilities. TrueOS users are encouraged to reference this guide, as the SysAdm client is included with TrueOS.

3. **Server Handbook** (https://sysadm.us/handbook/server/introduction.html)

The Server handbook is a basic installation guide, walking new users through the process of initializing SysAdm with a bridge and server connection.

## Filing Issues or Feature Requests <a name="fileissues"></a>

Due to the number of repositories under the TrueOS "umbrella", the TrueOS Project consolidates its issue trackers into a few repositories:

* [trueos-core](https://github.com/trueos/trueos-core) : Used for general TrueOS issues, Pico issues, and feature  requests.
* [lumina](https://github.com/trueos/lumina) : Issues related to using the Lumina Desktop Environment.
* (Coming Soon) [sysadm](https://github.com/trueos/sysadm) : Issues with using the SysAdm client or server.
* [trueos-docs](https://github.com/trueos/trueos-docs) : Issues related to the TrueOS Handbook.
* [lumina-docs](https://github.com/trueos/lumina-docs) : Issues related to the Lumina Handbook.
* [sysadm-docs](https://github.com/trueos/sysadm-docs) : Issues related to the SysAdm API Guide, Client, and Server Handbooks.
* [trueos-website](https://github.com/trueos/trueos-website) : Issues involving any of the TrueOS Project websites: 
  - https://www.lumina-desktop.org
  - https://www.trueos.org
  - https://www.sysadm.us

The TrueOS handbook has detailed instructions to help you report a bug (https://www.trueos.org/handbook/helpsupport.html#report-a-bug). It is recommended to refer to these instructions when creating new GitHub issues. Better bug reports usually result in faster fixes!

To request a feature, open a new issue in one of the related GitHub issue repositories and begin the title with *Feature Request:*.

## Community Channels <a name="community"></a>

The TrueOS community has a wide variety of chat channels and forum options available for users to interact with not only each other, but contributors to the project and the core development team too.

### Discourse <a name="discourse"></a>

TrueOS  has a [Discourse channel](https://discourse.trueos.org/) managed concurrently with the TrueOS Subreddit. New users need to sign up with Discourse in order to create posts, but it is possible to view posts without an account.

### Gitter <a name="gitter"></a>

The TrueOS Project uses Gitter to provide real-time chat and collaboration with TrueOS users and developers. Gitter does not require an application to use, but does require a login using either an existing GitHub or Twitter account.

To access the TrueOS Gitter community, point a web browser to https://gitter.im/trueos.

Gitter also maintains a full archive of the chat history. This means lengthy conversations about hardware issues or workarounds are always available for reference. To access the Gitter archive, navigate to the desired TrueOS room’s archive. For example, here is the address of the TrueOS Lobby archive: https://gitter.im/trueos/Lobby/archives.

### IRC <a name="irc"></a>

Like many open source projects, TrueOS has an Internet Relay Chat (IRC) channel so users can chat and get help in real time. To get connected, use this information in your IRC client:

* Server name: irc.freenode.net
* Channel name: #trueos (note the # is required)

### Subreddit <a name="subreddit"></a>

The TrueOS Project also has a [Subreddit](https://www.reddit.com/r/TrueOS/) for users who prefer to use Reddit to ask questions and to search for or post how-tos. A Reddit account is not required in order to read the Subreddit, but it is necessary to create a login account to submit or comment on posts.

## Social Media <a name="socialmedia"></a>

The TrueOS Project also maintains a number of social media accounts you can watch:

* Facebook: https://www.facebook.com/groups/4210443834/
* Linkedin: http://www.linkedin.com/groups?gid=1942544
* TrueOS Blog: https://www.trueos.org/blog/
* Twitter: https://twitter.com/TrueOS_Project/

