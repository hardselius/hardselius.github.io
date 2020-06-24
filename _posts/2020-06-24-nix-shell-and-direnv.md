---
layout: post
title: "Heroes in a nix-shell, direnv power!"
description: "Development environment with nix-shell and direnv"
comments: false
keywords: "nix, direnv"
---

In this post I'll walk through seting up Nix with `direnv` to automatically
provision a development environment with Nix Shell. I recently learned about
this from a colleague of mine and wanted to try it out. While doing so, I found
the [official documentation][2] on how to do it, which kind of makes this post
redundant. But I'm mostly writing this for myself, so what the heck. Here goes.

## Installing `direnv`

I'm using Nix Darwin, so I'll start by editing my `darwin-configuration.nix` to
add `direnv`.

```
  environment.systemPackages = [
    ...
    pkgs.direnv
    ...
  ];
```

Cool. Let's go ahead and install that.

```
$ darwin-rebuild switch
```

Now it's time to hook `direnv` into our shell. Head over to the [direnv
website][1] to find instructions for how to do it in your shell. Personally,
I'm using zsh which means I've got to add

```
eval "$(direnv hook zsh)"
```

at the end of my `.zshrc`. 

Once that's done, let's try out the demo from the [direnv website][1].

```
# Create a new folder for demo purposes.
$ mkdir ~/my-project
$ cd ~/my-project

# Show that the FOO environment variable is not loaded.
$ echo ${FOO-nope}
nope

# Create a new .envrc. This file is bash code that is going to be loaded by
# direnv.
$ echo export FOO=foo > .envrc
.envrc is not allowed

# The security mechanism didn't allow to load the .envrc. Since we trust it,
# let's allow it's execution.
$ direnv allow .
direnv: reloading
direnv: loading .envrc
direnv export: +FOO

# Show that the FOO environment variable is loaded.
$ echo ${FOO-nope}
foo

# Exit the project
$ cd ..
direnv: unloading

# And now FOO is unset again
$ echo ${FOO-nope}
nope
```

So far so good.

## Nix shell

Now let's create a `shell.nix` in the same project we created from the demo,
right alongside the `.envrc`. Maybe we want it to be a go project.

```
{ pkgs ? import <nixpkgs> {} }:
  pkgs.mkShell {
    buildInputs = [
      pkgs.go
    ];
  }
```

Write the file, and let's edit our `.envrc` by sticking `use_nix` in it. The
contents of your `.direnv` now look like this

```
use_nix
export FOO=foo
```

Once again, `direnv` is not allowed. We have to allow it again

```
$ direnv allow
```

That's kind of annoying, but once we allow it you should see some exciting
stuff happeing. Go is being installed! Once the installation finishes, let's
have a look

```
# Show which go binary is currently on the $PATH
$ which go
/nix/store/hzglx2vc1dziv8zxr9kxr8x4bgyanwnk-go-1.14.3/bin/go

# Back out of the project directory
$ cd ..
direnv: unloading

# Check the binary again (I'm expecting this to show the one installed with
# Homebrew)
$ which go
/usr/local/bin/go

```

That's pretty awesome! We can also move the environment variable from our
`.envrc` into `shell.nix`.

```
{ pkgs ? import <nixpkgs> {} }:
  pkgs.mkShell {
    buildInputs = [
      pkgs.go
    ];

    FOO="foo";
  }
```

When we're keeping everything in our `shell.nix` there's no longer any need for
running `direnv allow` after every change.

## Next steps

This is a great way to create a reproducible development environment, but
certainly the rabbit hole goes deeper. In [How I Start: Nix][5], Christine
Dodrill describes how to use [lorri][3] and [niv][4] to help manage the
development shell and project dependencies. I'm excited to try these tools out
and maybe write about them in a future post.

## References

- [direnv][1]
- [Development environment with Nix Shell][2]

[1]: https://direnv.net/
[2]: https://nixos.wiki/wiki/Development_environment_with_nix-shell
[3]: https://github.com/target/lorri
[4]: https://github.com/nmattia/niv
[5]: https://christine.website/blog/how-i-start-nix-2020-03-08
