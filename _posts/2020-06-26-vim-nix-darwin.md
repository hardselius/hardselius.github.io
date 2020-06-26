---
layout: post
title: "Configure Vim for Darwin in nix-darwin"
description: "Configure Vim for Darwin in nix-darwin, or how to get the unnamed register to work"
comments: false
keywords: "nix, nix-darwin, darwin, vim"
---

In a [previous post](../nix-please) I explored installing Nix and
nix-darwin. The first program I installed was Vim though
`nixpkgs.vim_configurable` in order to get `+python3` support. I quickly
realized that access to the system clipboard was gone. My setting

```
set clipboard^=unnamed
```

no longer did me any favors. So I did some digging.

```
$ vim --version
...
+acl               -farsi             +mouse_sgr         +tag_binary
+arabic            +file_in_path      -mouse_sysmouse    -tag_old_static
+autocmd           +find_in_path      +mouse_urxvt       -tag_any_white
+autochdir         +float             +mouse_xterm       -tcl
-autoservername    +folding           +multi_byte        +termguicolors
+balloon_eval      -footer            +multi_lang        +terminal
+balloon_eval_term +fork()            -mzscheme          +terminfo
+browse            +gettext           +netbeans_intg     +termresponse
++builtin_terms    -hangul_input      +num64             +textobjects
+byte_offset       +iconv             +packages          +textprop
+channel           +insert_expand     +path_extra        +timers
+cindent           +ipv6              -perl              +title
+clientserver      +job               +persistent_undo   +toolbar
+clipboard         +jumplist          +popupwin          +user_commands
+cmdline_compl     +keymap            +postscript        +vartabs
+cmdline_hist      +lambda            +printer           +vertsplit
+cmdline_info      +langmap           +profile           +virtualedit
+comments          +libcall           -python            +visual
+conceal           +linebreak         +python3           +visualextra
+cryptv            +lispindent        +quickfix          +viminfo
+cscope            +listcmds          +reltime           +vreplace
+cursorbind        +localmap          +rightleft         +wildignore
+cursorshape       +lua               +ruby              +wildmenu
+dialog_con_gui    +menu              +scrollbind        +windows
+diff              +mksession         +signs             +writebackup
+digraphs          +modify_fname      +smartindent       +X11
+dnd               +mouse             -sound             -xfontset
-ebcdic            +mouseshape        +spell             +xim
+emacs_tags        +mouse_dec         +startuptime       +xpm
+eval              -mouse_gpm         +statusline        -xsmp
+ex_extra          -mouse_jsbterm     -sun_workshop      +xterm_clipboard
+extra_search      +mouse_netterm     +syntax            -xterm_save
...
```

As it turns out, Vim was also compiled with `+X11` and `+xterm_clipboard`. Aha!
So Vim uses the X11 selection mechanism for copy and paste instead of macOS'
system clipboard. That kind of sucks. So how to we get Vim to interoperate with
the `*` or `+` registers?

We could of course install [XQuartz][xquartz] and configure it to "Update
Pasteboard immediately when new text is selected". That'll do the trick. But
what if XQuartz is not running when we first start Vim? It will start
automatically, but we get an unnacceptable start time. Waiting for several
seconds in addition to having to install yet another thing I don't really want
is not the way to go. There has to be another way.


Looking at the [source][configurable.nix] for `vim_configurable` we see

```
, features          ? "huge" # One of tiny, small, normal, big or huge
, wrapPythonDrv     ? false
, guiSupport        ? config.vim.gui or (if stdenv.isDarwin then "gtk2" else "gtk3")
, luaSupport        ? config.vim.lua or true
, perlSupport       ? config.vim.perl or false      # Perl interpreter
, pythonSupport     ? config.vim.python or true     # Python interpreter
, rubySupport       ? config.vim.ruby or true       # Ruby interpreter
, nlsSupport        ? config.vim.nls or false       # Enable NLS (gettext())
, tclSupport        ? config.vim.tcl or false       # Include Tcl interpreter
, multibyteSupport  ? config.vim.multibyte or false # Enable multibyte editing support
, cscopeSupport     ? config.vim.cscope or true     # Enable cscope interface
, netbeansSupport   ? config.netbeans or true       # Enable NetBeans integration support.
, ximSupport        ? config.vim.xim or true        # less than 15KB, needed for deadkeys
, darwinSupport     ? config.vim.darwin or false    # Enable Darwin support
, ftNixSupport      ? config.vim.ftNix or true      # Add .nix filetype detection and minimal syntax highlighting support
, ...
```

`guiSupport` populates the `--enable-gui` flag and `darwinSupport` (`false` by
default) will set the `--disable-darwin` flag. When `darwinSupport` is set to
`true`, the `--enable-darwin` flag is set instead. Let's edit our `darwin-configuration.nix`

```
  ...
  
  environment.systemPackages = [
    ...
    config.programs.vim.package
    ...
  ];

  ...

  programs.vim.package = pkgs.vim_configurable.override {
    python = pkgs.python3;
    guiSupport = "no";
    darwinSupport = true;
  };

  ...
```

Then we `$ darwin-rebuild switch` and wait for the compilation to finish.

```
$ vim --version
...
+acl               -farsi             +mouse_sgr         +tag_binary
+arabic            +file_in_path      -mouse_sysmouse    -tag_old_static
+autocmd           +find_in_path      +mouse_urxvt       -tag_any_white
+autochdir         +float             +mouse_xterm       -tcl
-autoservername    +folding           +multi_byte        +termguicolors
-balloon_eval      -footer            +multi_lang        +terminal
+balloon_eval_term +fork()            -mzscheme          +terminfo
-browse            +gettext           +netbeans_intg     +termresponse
++builtin_terms    -hangul_input      +num64             +textobjects
+byte_offset       +iconv             +packages          +textprop
+channel           +insert_expand     +path_extra        +timers
+cindent           +ipv6              -perl              +title
-clientserver      +job               +persistent_undo   -toolbar
+clipboard         +jumplist          +popupwin          +user_commands
+cmdline_compl     +keymap            +postscript        +vartabs
+cmdline_hist      +lambda            +printer           +vertsplit
+cmdline_info      +langmap           +profile           +virtualedit
+comments          +libcall           -python            +visual
+conceal           +linebreak         +python3           +visualextra
+cryptv            +lispindent        +quickfix          +viminfo
+cscope            +listcmds          +reltime           +vreplace
+cursorbind        +localmap          +rightleft         +wildignore
+cursorshape       +lua               +ruby              +wildmenu
+dialog_con        +menu              +scrollbind        +windows
+diff              +mksession         +signs             +writebackup
+digraphs          +modify_fname      +smartindent       -X11
-dnd               +mouse             -sound             -xfontset
-ebcdic            -mouseshape        +spell             -xim
+emacs_tags        +mouse_dec         +startuptime       -xpm
+eval              -mouse_gpm         +statusline        -xsmp
+ex_extra          -mouse_jsbterm     -sun_workshop      -xterm_clipboard
+extra_search      +mouse_netterm     +syntax            -xterm_save
...
```

Boom! `-X11` and `-xterm_clipboard` is exactly what we want to see. We now have
access to the system clipboard again and everything is back to normal.

[xquartz]: https://www.xquartz.org/
[configurable.nix]: https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/vim/configurable.nix
