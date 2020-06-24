---
layout: post
title: "Nix Please!"
description: "Replacing Homebrew with Nix Package Manager and Nix Darwin"
comments: false
keywords: "nix, homebrew, vim"
---

After reading a [bunch][1] of [stuff][2] on the internet of how to replace
[Homebrew][3] with [Nix][4], I decided to finally take the plunge. This post
outlines my efforts to get up and running with the Nix package manager and [Nix
Darwin][5].

## Installing Nix

Nix gives you two installation options; single user or multi-user. Let's go
with the recommended multi-user install.

Executing the following

```
$ sh <(curl -L https://nixos.org/nix/install) --daemon
```

gives me the following output

```
...
Note: a multi-user installation is possible. See https://nixos.org/nix/manual/#sect-multi-user-installation

Installing on macOS >=10.15 requires relocating the store to an apfs volume.
Use sh <(curl https://nixos.org/nix/install) --darwin-use-unencrypted-nix-store-volume or run the preparation steps manually.
See https://nixos.org/nix/manual/#sect-macos-installation
```

Starting with macOS 10.15 (Catalina), the root filesystem is read-only. This
means that `/nix` can no longer live on you system volume. Thankfully, there
exists a workaraound for this. Instead of installing Nix using the command we
just executed, we run 

```
$ sh <(curl https://nixos.org/nix/install) --darwin-use-unencrypted-nix-store-volume
```

This should create an unencrypted APFS volume for your Nix store and a
"synthetic" empty directory to mount it over at `/nix`.

I actually had some trouble executing the script using `curl` and I didn't feel
like finding out why so I decided to just `wget` it and then run it. Anyways.
Progress.

```
     ------------------------------------------------------------------ 
    | This installer will create a volume for the nix store and        |
    | configure it to mount at /nix.  Follow these steps to uninstall. |
     ------------------------------------------------------------------ 

  1. Remove the entry from fstab using 'sudo vifs'
  2. Destroy the data volume using 'diskutil apfs deleteVolume'
  3. Remove the 'nix' line from /etc/synthetic.conf or the file
  ...
```

This installer will go on and ask for sudo password and do it's thing.

## Installing Nix Darwin

####  Backup a few files first

```
$ sudo mv /etc/zprofile /etc/zprofile.orig
$ sudo mv /etc/nix/nix.conf /etc/nix/nix.conf.orig
$ sudo mv /etc/zshrc /etc/zshrc.orig
```

#### Install Nix Darwin

```
$ nix-build https://github.com/LnL7/nix-darwin/archive/master.tar.gz -A installer
$ ./result/bin/darwin-installer
```

During the installation, we're presented with some options.

```
Would you like edit the default configuration.nix before starting? [y/n] n
Would you like to manage <darwin> with nix-channel? [y/n] y
Would you like to load darwin configuration in /etc/bashrc? [y/n] y
Would you like to load darwin configuration in /etc/zshrc? [y/n] y
Would you like to create /run? [y/n] y
```

Nice. The installation finishes up and gives us some helpful instructions

```
    Open '/Users/martin/.nixpkgs/darwin-configuration.nix' to get started.
    See the README for more information: https://github.com/LnL7/nix-darwin/blob/master/README.md

    Don't forget to start a new shell or source /etc/static/bashrc.
```

## Installing packages

Now that we have `nix-darwin` installed, we can start installing packages.

### Vim

Naturally, I want to start by installing `vim`. Let's look for it

```
$ nix-env -qaP vim
```

The result of the query is

```
nixpkgs.vim  vim-8.2.0701
```

Alright. So I guess we just add it to our `darwin-configuration.nix` then.

```
  environment.systemPackages = [
    pkgs.vim
  ];
```

Save, run `darwin-rebuild switch`, and wait for the result. Once the
installation finishes we should have Vim installed.

Depending on how your `PATH` is set up, and if you installed `vim` with
`homebrew`, you might still be on your previous version. Go ahead and check

```
$ which vim
# Alt 1: /usr/local/bin/vim
# Alt 2: /run/current-system/sw/bin/vim
```

If you get something like `Alt 1`, you can run `brew unlink vim` and log in
again. Run `which vim` again and you should get something that looks like the
second path.

When I fired up Vim I quickly realized that the package wasn't compiled with
`+python3`. In my personal vim setup I have a plugin that depends on that, so I
needed to figure out how to solve that issue. After I did some digging around,
I found that there's a Nix package callen `vim_configurable` that I could use.

To install a Vim with Python 3 support, we need to edit our
`darwin-configuration.nix` a little bit.

```
{
  ...

  environment.systemPackages = [
    config.programs.vim.package
  ];

  ...

  programs.vim.package = pkgs.vim_configurable.override {
    python = pkgs.python3
  };

  ...
}
```

Now run

```
$ darwin-rebuild switch
```

again, and you should see Vim being compiled. This takes a while, but once it
finishes up you have Vim working with `+python3`. Neat!

## Moving the rest of the packages

Moving the rest of the packages should be a pretty straight-forward process. Up
until now, I have used a `Brewfile` to keep inventory of all installed
packages.

To update my `Brewfile` with everything that's installed using Homebrew, I run

```
$ brew bundle dump --force
```

Then I move packages from Homebrew into `darwin-configuration.nix`, as
described in Sala Rahmainan's [article][1].

Occasionally, I run

```
$ brew bundle cleanup --force
```

to remove any garbage left from uninstalled Homebrew packages.

## References

- [Moving from Homebrew to Nix Package Manager][1] (Salar Rahmanian)
- [Use Nix on maxOS as a Homebrew User][2] (Yufan Lou)
- [Homebrew][3]
- [NixOS][4]
- [Nix Darwin][5]
- [Install Nix][6]
- [Nix macOS installation][7]

[1]: https://www.softinio.com/post/moving-from-homebrew-to-nix-package-manager/
[2]: https://dev.to/louy2/use-nix-on-macos-as-a-homebrew-user-22d
[3]: https://brew.sh/
[4]: https://nixos.org/
[5]: https://github.com/LnL7/nix-darwin
[6]: https://nixos.org/guides/install-nix.html
[7]: https://nixos.org/nix/manual/#sect-macos-installation
