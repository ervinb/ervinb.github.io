---
layout: post
title: Prevent systemd installation on Ubuntu 14.04 and other Debian based distros
tags: linux systemd tips
---

"If the mountain won't come to Muhammad then Muhammad must go to the mountain."

Oddly, this proverb applies to `systemd` as well. It's rapidly making its way
to all the major distros. Love it, or hate it, it's here to stay.

The thing that throws me off the most, is its intrusiveness. When you install `systemd`,
it literally plows through the wall, and doesn't bother with the door.
This is especially true if you are running an older Ubuntu, like 14.04. As packages
are rapidly adopting `systemd` as a dependency, there's a good chance that it will
surprise you with a visit. Sooner than you think.

I had a minimalistic Ubuntu Trusty environment, and upon updating
Java to `8u121`, `systemd` joined the party as well, breaking SSH and some other
crucial services. It also drank all the beer. Java version `8u111` didn't have
it as an indirect dependency.

Here's how to prevent `systemd` from ruining your party:

```sh
# create an APT config file
$ sudo vi /etc/apt/preferences.d/systemd
# paste this snippet and save -- make sure there's no indentation
Package: systemd
Pin: release *
Pin-Priority: -1
# at this point, the walls are reinforced and the beer is restocked
```

In a nutshell, by setting the `Priority` to a negative number, APT installations
will skip the `systemd` package, even if it is a dependency. Check out the
[apt_preferences](https://linux.die.net/man/5/apt_preferences) man page for more
info. I will probably revisit it in another post.

Alternatively, the same can be achieved with the `--no-install-recommends`
option for `apt-get install`, when `systemd` isn't a direct dependency, but then you
might miss out on some valuable packages, like `dbus` in my case.
You can broaden the match by putting an asterisk (`*`) around the package name,
but in this case, I think it's an overkill. You may want to install [timedatectl](http://manpages.ubuntu.com/manpages/trusty/man1/timedatectl.1.html)
without installing `systemd`, which is part of the [systemd-services](https://launchpad.net/ubuntu/trusty/+package/systemd-services)
package in Trusty. With limiting our match to `systemd` only, the
`systemd-services` package can happily bring in `timedatectl` without reaping havoc.

Done.
