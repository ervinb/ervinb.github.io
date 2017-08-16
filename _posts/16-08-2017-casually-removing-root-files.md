---
layout: post
title: Casually removing root files
tags: linux tips
---

You're walking at `$HOME`, minding your own business.

```
$ whoami
> user

$ pwd
> /home/user
```

But something is bothering your feet. It's like if a little rock has fallen into your shoe.
You take it off, to see what's going on.

```
$ ls -lah ./left-shoe/little-rock
---------- 1 root root 4 May 30 13:20 little-rock
```

That's odd. It's there, but it doesn't seem to be yours. It's left there by
`root`, the Rock Tamer, and only he can decide its fate.

```
# bash -c "echo 'You stay here' > /home/user/shoe/little-rock"
# chmod 0000 /home/user/left-shoe/little-rock
```

You reach into your pocket for your phone, to speed dial him with `sudo`. Suddenly,
you feel powerful (from watching Gladiator last night), and decide to put back
the phone, and try your luck.

```
$ rm -f ./left-shoe/little-rock
$ ls -lah ./left-shoe/little-rock
ls: cannot access little-rock: No such file or directory
```

You look down at your shaking hands, trying to figure out if this is the real world.
It is. You did it. Without the Rock Tamer. But how?

The little rock in your shoe had absolutely no idea what's coming. As seen from
it's incarnation, nobody had [any permissions](http://linuxcommand.org/lts0070.php)
on it (`--- --- ---`). No reads, no writes, no throwing by anyone (owner, group, others).

## The catch

What happened is, is that the Rock Tamer forgot that you are even more powerful
than him, when you're at `$HOME`. Let's see why.

To be able to do anything with a file, the first step is to look it up. Listing a directory's
contents is controlled by the execute flag. If a user has execute permissions on a directory,
he can see what's inside it. Also, the execute flag gives access to the file's
`inode`, which is crucial in this context, as the removal process [unlinks](https://linux.die.net/man/2/unlinkat) the file.

Next, the removing part. Renaming or removing a file doesn't involve the `write()` system call.
Practically, we don't need any permissions to remove the file, nor do we care
about its owner. The only requirement is to have write permissions on the parent directory (and
the execute flag on the parent directory).

The `$HOME` directory naturally fulfills both of these requirements from the user's perspective.


## The contra-catch

If the Rock Tamer, really didn't want anyone to mess around with his rocks, he would've done:

```
$ chattr +i /home/user/left-shoe/little-rock
```

This operation makes the file immutable, which among other things, prevents its removal.
Excerpt from the [man page](https://linux.die.net/man/1/chattr):

```
A file with the 'i' attribute cannot be modified: it cannot be deleted or renamed, no link can be created to this file and no data can be written to the file. Only the superuser or a process possessing the CAP_LINUX_IMMUTABLE capability can set or clear this attribute.
```

*Moonwalks away.*
