# Terminal Colors

_(Previously published and discussed at https://gist.github.com/XVilka/8346728.)_

There exists common confusion about terminal colors. This is what we have right now:

- Plain ASCII
- ANSI escape codes: 16 color codes with bold/italic and background
- 256 color palette: 216 colors + 16 ANSI + 24 gray (colors are 24-bit)
- 24-bit truecolor: "888" colors (aka 16 million)

```bash
printf "\x1b[${bg};2;${red};${green};${blue}m\n"
```

The 256-color palette is configured at start and is a 666-cube of colors,
each of them defined as a 24-bit (888 RGB) color.

This means that current support can only display 256 different colors in the
terminal while "truecolor" means that you can display 16 million different
colors at the same time.

Truecolor escape codes do not use a color palette. They just specify the
color directly.

This is a good test case:

```bash
printf "\x1b[38;2;255;100;0mTRUECOLOR\x1b[0m\n"
```

- or
  https://raw.githubusercontent.com/JohnMorales/dotfiles/master/colors/24-bit-color.sh
- or https://github.com/robertknight/konsole/tree/master/tests/color-spaces.pl
- or https://git.gnome.org/browse/vte/tree/perf/img.sh
- or just run this:

```sh
awk 'BEGIN{
    s="/\\/\\/\\/\\/\\"; s=s s s s s s s s;
    for (colnum = 0; colnum<77; colnum++) {
        r = 255-(colnum*255/76);
        g = (colnum*510/76);
        b = (colnum*255/76);
        if (g>255) g = 510-g;
        printf "\033[48;2;%d;%d;%dm", r,g,b;
        printf "\033[38;2;%d;%d;%dm", 255-r,255-g,255-b;
        printf "%s\033[0m", substr(s,colnum+1,1);
    }
    printf "\n";
}'
```

Keep in mind that it is possible to use both ';' and ':' as Control Sequence
delimiters.

According to Wikipedia[1], this behavior is only supported by xterm and konsole.

[1] https://en.wikipedia.org/wiki/ANSI_color

# Truecolor Detection

## Checking for COLORTERM

[VTE](https://bugzilla.gnome.org/show_bug.cgi?id=754521),
[Konsole](https://bugs.kde.org/show_bug.cgi?id=371919) and
[iTerm2](https://gitlab.com/gnachman/iterm2/issues/5294) all advertise
truecolor support by placing `COLORTERM=truecolor` in the environment of the
shell user's shell. This has been in VTE for a while, but is relatively new in
Konsole and iTerm2 and has to be enabled at compile time (most packages do not,
so you have to compile them yourself from the git source repo).

The S-Lang library has a check that `$COLORTERM` contains either "truecolor" or
"24bit" (case sensitive).

Terminfo has supported the 24-bit TrueColor capability since
[ncurses-6.0-20180121](https://lists.gnu.org/archive/html/bug-ncurses/2018-01/msg00045.html),
under the name "RGB".
You need to use the "setaf" and "setab" commands to set the foreground and
background respectively.

Having an extra environment variable (separate from `TERM`) is not ideal: by
default it is not forwarded via sudo, ssh, etc, and so it may still be
unreliable even where support is available in programs. (It does however err on
the side of safety: it does not advertise support when it is not actually
supported, and the programs should fall back to using 8-bit color.)

These issues can be ameliorated by adding `COLORTERM` to:
* the `SendEnv` list in `/etc/ssh/ssh_config` on ssh clients;
* the `AcceptEnv` list in `/etc/ssh/sshd_config` on ssh servers; and
* the `env_keep` list in `/etc/sudoers`.

Despite these problems, it's currently the best option, so checking
`$COLORTERM` is recommended since it will lead to a more seamless desktop
experience where only one variable needs to be set.

App developers can freely choose to check for this variable, or introduce their
own method (e.g. an option in their config file). They should use whichever
method best matches the overall design of their app.

Ideally any terminal that really supports truecolor would set this variable;
but as a work-around you might need to put a check in `/etc/profile` to set
`COLORTERM=truecolor` when `$TERM` matches any terminal type known to have
working truecolor.

```lang=sh
case $TERM in
  iterm            |\
  linux-truecolor  |\
  screen-truecolor |\
  tmux-truecolor   |\
  xterm-truecolor  )    export COLORTERM=truecolor ;;
  vte*)
esac
```

## Querying The Terminal

In an interactive program that can read terminal responses, a more reliable
method is available, that is transparent to sudo & ssh.

Simply try sending a truecolor value to the terminal, followed by a query to
ask what color it currently has. If the response indicates the same color as
was just set, then truecolor is supported.

If the response indicates an 8-bit color, or does not indicate a color, or if
no response is forthcoming within a few centiseconds, then assume that
truecolor is not supported.

```bash
$ (echo -e '\e[48:2:1:2:3m\eP$qm\e\\' ; xxd)

^[P1$r48:2:1:2:3m^[\
00000000: 1b50 3124 7234 383a 323a 313a 323a 336d  .P1$r48:2:1:2:3m
```

Here we set the background color to `RGB(1,2,3)` - an unlikely default
choice - and request the value that we just set. The response comes back that
the request was understood (`1`), and that the color is indeed `48:2:1:2:3`.
This tells us also that the terminal supports the colon delimiter. If instead,
the terminal did not support truecolor we might see a response like

```
^[P1$r40m^[\
00000000: 1b50 3124 7234 306d 1b5c 0a              .P1$r40m.\.
```

This terminal replied that the color is `40` - it has not accepted our request
to set `48:2:1:2:3`.

```
^[P0$r^[\
00000000: 1b50 3024 721b 5c0a                      .P0$r.\.
```

This terminal did not even understand the `DECRQSS` request - its response was
`CSI`+`0$r`. This does not indicate whether it set the color, but since it
doesn't understand how to reply to our request it is unlikely to support
truecolor either.

# Truecolor Support in Output Devices

## Fully Supporting

### Terminal Emulators

- [xterm](https://invisible-island.net/xterm/) - (from
  [331 (change 330j)](https://invisible-island.net/xterm/xterm.log.html#xterm_331)
- [st](https://st.suckless.org/) (from suckless) [delimiter: semicolon] -
  https://lists.suckless.org/dev/1307/16688.html
- [xst](https://github.com/gnotclub/xst) - fork of st
- [konsole](https://konsole.kde.org/) [delimiter: colon,
  semicolon] - https://bugs.kde.org/show_bug.cgi?id=107487
- [iTerm2](https://iterm2.com/) [delimiter: colon, semicolon] - since v3
  version
- [Therm](https://github.com/trufae/Therm) [delimiter: colon, semicolon] - fork
  of iTerm2
- [qterminal](https://github.com/lxqt/qterminal) [delimiter: semicolon] - after version 0.14.1 ([issue #78](https://github.com/qterminal/qterminal/issues/78))
- [alacritty](https://github.com/jwilm/alacritty) [delimiter: semicolon] -
  written in Rust
- [Contour](https://github.com/contour-terminal/contour) [delimiter: semicolon] - written in C++17, uses OpenGL
- [kitty](https://github.com/kovidgoyal/kitty) [delimiter: colon,semicolon] -
  uses OpenGL
- [foot](https://codeberg.org/dnkl/foot) [delimiter: colon, semicolon] -
  Wayland terminal
- [cool-retro-term](https://github.com/Swordfish90/cool-retro-term) [delimiter:
  semicolon]
- [mosh](https://mosh.org/) (Mobile SHell) [delimiter: semicolon] - since commit [6cfa4aef598146cfbde7f7a4a83438c3769a2835](https://github.com/mobile-shell/mosh/commit/6cfa4aef598146cfbde7f7a4a83438c3769a2835)
- [pangoterm](http://www.leonerd.org.uk/code/pangoterm/) [delimiter:
  colon, semicolon]
- [Termux](https://termux.com/) [delimiter: semicolon] - **Android platform**
- [ConnectBot](https://connectbot.org/) - **Android platform** - since [3bcc75ccedaf2136b04c5932c81a5155f29dc3b5](https://github.com/connectbot/connectbot/commit/3bcc75ccedaf2136b04c5932c81a5155f29dc3b5) commit.
- [Black Screen](https://github.com/shockone/black-screen) [delimiter:
  semicolon] - cross-platform, HTML/CSS/JS-based
- [hterm](https://chromium.googlesource.com/apps/libapps/+/master/hterm) -
  HTML/CSS/JS-based (ChromeOS)
- [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/) -
  [landed](https://git.tartarus.org/?p=simon/putty.git;a=commit;h=a4cbd3dfdb71d258e83bbf5b03a874c06d0b3106)
  in git (patched version [3] {approximation to 256 colors} and [4]
  {real truecolors} available) - **Windows platform**
- [Tera Term](http://en.sourceforge.jp/projects/ttssh2/) [delimiter: colon,
  semicolon] - **Windows platform**
- [ConEmu](https://github.com/Maximus5/ConEmu) [delimiter: semicolon] -
  **Windows platform**
- [Windows
  Powershell](https://en.wikipedia.org/wiki/PowerShell#PowerShell_5.1)
  [delimiter: semicolon] - aka PowerShell 5.x and below **Windows 10**
- [PowerShell Core](https://github.com/PowerShell/PowerShell) [delimiter:
  semicolon] aka PowerShell 6+ **Windows 10**
- [cmd.exe](https://en.wikipedia.org/wiki/Cmd.exe) [delimiter:
  semicolon] Built-in Windows shell that is mostly unchanged since DOS **Windows 10**
- [FinalTerm](http://finalterm.org/) [delimiter: semicolon] -
  **[abandoned](https://worldwidemann.com/finally-terminated/)**, iTerm2
  [borrowing it's ideas and features](https://iterm2.com/documentation-shell-integration.html).
- [MacTerm](https://github.com/kmgrant/macterm) [delimiter: semicolon] - **Mac
  OS X platform**
- [mintty](https://mintty.github.io/) [delimiter: semicolon] **Cygwin and
  MSYS/MSYS2** since commit [43f0ed8a46c6549cb9a3ea27abc057b5abe13bdb](https://github.com/mintty/mintty/commit/43f0ed8a46c6549cb9a3ea27abc057b5abe13bdb)
  (2.0.1 release) - **Windows platform**
- [MobaXterm](https://mobaxterm.mobatek.net/) **Windows platform** - closed
  source (run `lscolors` to see a truecolor test)
- [ZOC](https://www.emtec.com/zoc/index.html) **Windows/OS X platform** - closed
  source since
  [7.19.0 version](https://www.emtec.com/downloads/zoc/zoc_changes.txt)
- [upterm](https://github.com/railsware/upterm) *Windows/MacOS/Linux Electron* -
  A terminal emulator for the 21st century.
- Windows 10 bash console, since
  [Windows Insiders build 14931](https://blogs.msdn.microsoft.com/commandline/2016/09/22/24-bit-color-in-the-windows-console/)
- All [libvte](https://download.gnome.org/sources/vte/) based terminals
  (since 0.36 version) [delimiter: colon, semicolon] -
  https://bugzilla.gnome.org/show_bug.cgi?id=704449
  - **libvte**-based
    [Gnome Terminal](https://help.gnome.org/users/gnome-terminal/stable/)
  - **libvte**-based [sakura](https://www.pleyades.net/david/projects/sakura)
  - **libvte**-based
    [xfce4-terminal](https://docs.xfce.org/apps/terminal/start) - since
    [0.6.90](https://github.com/xfce-mirror/xfce4-terminal/releases/tag/xfce4-terminal-0.6.90)
    release, if compiled with GTK+3
  - **libvte**-based
    [Terminator](https://gnometerminator.blogspot.com/p/introduction.html) -
    since [1.90](https://launchpad.net/terminator/+announcement/14358) release
  - **libvte**-based [Tilix](https://github.com/gnunn1/tilix) - written in D.
    Similar user interface as for Terminator.
  - **libvte**-based [Lilyterm](https://lilyterm.luna.com.tw/) - since commit [72536e7ba448ad9ef1126ce45fbde3a3407a271b](https://github.com/Tetralet/LilyTerm/commit/72536e7ba448ad9ef1126ce45fbde3a3407a271b)
  - **libvte**-based [ROXTerm](http://roxterm.sourceforge.net/)
  - **libvte**-based [evilvte](https://www.calno.com/evilvte/) - no release yet,
    version from git https://github.com/caleb-/evilvte
  - **libvte**-based [Termit](https://github.com/nonstop/termit)
  - **libvte**-based [Termite](https://github.com/thestinger/termite) ([NOT MAINTAINED](https://github.com/thestinger/termite/issues/760))
  - **libvte**-based [Tilda](https://github.com/lanoxx/tilda)
  - **libvte**-based [tinyterm](https://code.google.com/p/tinyterm)
  - **libvte**-based
    [Pantheon Terminal](https://launchpad.net/pantheon-terminal)
  - **libvte**-based [lxterminal](https://sourceforge.net/projects/lxde) - with
    **--enable-gtk3** configure flag.
  - **libvte**-based [guake](http://guake-project.org/) - A top-down terminal for GNOME
- All [xterm.js](https://github.com/xtermjs/xterm.js) based terminals (since [v3.13](https://github.com/xtermjs/xterm.js/issues/484), [v4.3 for webgl](https://github.com/xtermjs/xterm.js/pull/2552)) [delimiter: semicolon]
  - [VS Code](https://code.visualstudio.com/)'s integrated terminal
  - [Tabby](https://github.com/Eugeny/tabby): highly configurable terminal emulator for Windows, macOS and Linux
  - [Hyper.app](https://hyper.is/): crossplatform, HTML/CSS/JS-based (Electron)
- [Netsarang XShell](https://www.netsarang.com/products/xsh_overview.html) -
  Xshell7/ Xshell6 >= Build 0181
  (You must set _**Tools-Options.. -Advanced**_, check the _**Use true color\***_ and **reopen** the software) 

There are a bunch of libvte-based terminals for GTK2, so they are listed in the
another section.

### Multiplexers

- [tmux](https://tmux.github.io/) - starting from version 2.2 (support since
  [427b820...](https://github.com/tmux/tmux/commit/427b8204268af5548d09b830e101c59daa095df9))
- [screen](https://git.savannah.gnu.org/cgit/screen.git/) - has support in
  'master' branch, need to be enabled (see 'truecolor' option)
- [pymux](https://github.com/jonathanslenders/pymux) - tmux clone in pure Python
  (to enable truecolor run pymux with `--truecolor` option)
- [dvtm](https://github.com/martanne/dvtm) - not yet supporting truecolor
  https://github.com/martanne/dvtm/issues/10

### Re-players

- [asciinema](https://asciinema.org/) player:
  https://github.com/asciinema/asciinema-player

## Partial Support

These terminal emulators parse ANSI color sequences, but approximate the true
color using a palette or limit number of true colors that can be used at the
same time. A 256-color (8-bit) palette is used unless specified.

- [mlterm](http://mlterm.sourceforge.net/) - built with **--with-gtk=3.0**
  configure flag. Approximates colors using a 512-color embedded palette
  (https://sourceforge.net/p/mlterm/bugs/74/)
- [urxvt aka rxvt-unicode](http://software.schmorp.de/pkg/rxvt-unicode.html) -
  since
  [revision 1.570](http://cvs.schmorp.de/rxvt-unicode/src/command.C?revision=1.570&view=markup&sortby=log&sortdir=down).
  Limits maximum number of colors:
  http://lists.schmorp.de/pipermail/rxvt-unicode/2016q2/002261.html
- Linux console ([fbcon](https://www.kernel.org/doc/html/latest/fb/fbcon.html)), [since v3.16](https://github.com/torvalds/linux/commit/cec5b2a97a11ade56a701e83044d0a2a984c67b4) - https://bugzilla.kernel.org/show_bug.cgi?id=79551 (downgraded to 16 foregrounds and 8 backgrounds)

#### Note about color differences

Human eyes are sensitive to the primary colors in such a way that the simple
Gaussian distance √(R²+G²+B²) gives poor results when trying to find the
"nearest" available color as perceived by most humans.

The [CIEDE2000](https://en.wikipedia.org/wiki/Color_difference#CIEDE2000)
formula provides much better perceptual matching, but it is considerably more
complex and may perform very slowly if used blindly [2].

[2] https://github.com/neovim/neovim/issues/793#issuecomment-48106948

## Not Supporting Truecolor

- [Terminal.app](https://en.wikipedia.org/wiki/Terminal_(macOS)): MacOS Terminal built-in
- [Terminology](https://www.enlightenment.org/about-terminology)
  (Enlightenment) - https://phab.enlightenment.org/T746
- [Hyper.app](https://hyper.is/) [delimiter: semicolon] - cross-platform,
  HTML/CSS/JS-based (Electron) https://github.com/zeit/hyper/issues/2294
- [Cmder](https://cmder.net/): Portable console emulator for Windows,
  based on ConEmu.
- [Terminus](https://github.com/Eugeny/terminus):
  highly configurable terminal emulator for Windows, MacOS and Linux
- [mrxvt](https://sourceforge.net/projects/materm) (looks abandoned) -
  https://sourceforge.net/p/materm/feature-requests/41/
- [aterm](http://www.afterstep.org/aterm.php) (looks abandoned) -
  https://sourceforge.net/p/aterm/feature-requests/23/
- [fbcon](https://www.kernel.org/doc/Documentation/fb/fbcon.txt) (prior to
  Linux 3.16) - https://bugzilla.kernel.org/show_bug.cgi?id=79551
- FreeBSD console - https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=191652
- [yaft](https://github.com/uobikiemukot/yaft) framebuffer terminal - [issue #12](https://github.com/uobikiemukot/yaft/issues/12)
- [KiTTY](https://www.9bis.net/kitty/) - **Windows platform**
- [MTPuTTY](https://ttyplus.com/) - **Windows platform**
- [mRemoteNG](https://mremoteng.org/) - **Windows platform** - [issue #717](https://github.com/mRemoteNG/mRemoteNG/issues/717)
- [JuiceSSH](https://juicessh.com/) - **Android platform**, closed source
- [Termius](https://www.termius.com/) - **Linux, Windows, OS X platforms**,
  closed source
- [SmarTTY](https://sysprogs.com/SmarTTY/) - **Windows platform** - closed source
  (sent them a request)
- libvte and GTK2 - based:
  - **libvte**-based [GTKTerm2](http://gtkterm.feige.net/)
  - **libvte**-based [stjerm](https://github.com/stjerm/stjerm) (looks abandoned) - [issue #39](https://github.com/stjerm/stjerm/issues/39)

# Console Programs + Truecolor

## Console Programs Supporting Truecolor

- [s-lang](https://lists.jedsoft.org/lists/slang-users/2015/0000020.html)
  library - (since pre2.3.1-35, for 64bit systems)
- [ncurses](https://www.gnu.org/software/ncurses/) library - since 6.1 version
- [Notcurses](https://notcurses.com/) library - all releases
- [Eternal Terminal](https://mistertea.github.io/EternalTCP/) - automatically
  reconnecting shell
- [mc](https://midnight-commander.org/) - since
  [682a5...](https://midnight-commander.org/changeset/682a5116edd20b8ba81743a1f7495c883b0ce644).
  See also [ticket #3724](https://midnight-commander.org/ticket/3724) for
  truecolor themes.
- [vifm](https://github.com/vifm/vifm) file manager - since 0.12 version
- [irssi](https://github.com/irssi/irssi) - since
  [PR #48](https://github.com/irssi/irssi/pull/48)
- [neovim](https://github.com/neovim/neovim) - since commit
  [8dd415e887923f99ab5daaeba9f0303e173dd1aa](https://github.com/neovim/neovim/commit/8dd415e887923f99ab5daaeba9f0303e173dd1aa);
  need to set
  [termguicolors](https://neovim.io/doc/user/options.html#%27termguicolors) to
  enable truecolor.
- [vim](https://github.com/vim/vim) - (from 7.4.1770); need to set
  [termguicolors](https://github.com/vim/vim/blob/master/runtime/doc/version8.txt#L202)
  to enable truecolor.
- [joe](https://sf.net/p/joe-editor) - (from
  [4.5](https://sourceforge.net/p/joe-editor/news/2017/09/joe-45-released/)
  version)
- [emacs](https://www.gnu.org/software/emacs/) - since
  [26.1 release](https://lists.gnu.org/archive/html/emacs-devel/2018-05/msg00765.html)
- [micro editor](https://micro-editor.github.io/)
- [dte](https://gitlab.com/craigbarnes/dte) text editor - (since
  [version 1.8](https://craigbarnes.gitlab.io/dte/releases.html#v1.8))
- [elinks](https://repo.or.cz/w/elinks.git) -
  [configure.in:1410](https://repo.or.cz/w/elinks.git/blob/HEAD:/configure.in#l1410)
  (./configure --enable-true-color)
- [tcell](https://github.com/gdamore/tcell) library for Go language
- [timg](https://github.com/hzeller/timg) - Terminal Image Viewer
- [tv](https://github.com/daleroberts/tv) - tool to quickly view high-resolution
  multi-band imagery directly in terminal
- [termimage](https://github.com/nabijaczleweli/termimage) - terminal image
  viewer
- [explosion](https://github.com/Tenzer/explosion) - terminal image viewer
- [ls-icons](https://github.com/sebastiencs/ls-icons) - fork of coreutils with
  `ls` program that supports icons
- [mpv](https://github.com/mpv-player/mpv) - video player with support of
  console-only output (since 0.22 version)
- [radare2](https://github.com/radareorg/radare2) - reverse engineering framework;
  since 0.9.6 version.
- [rizin](https://github.com/rizinorg/rizin) - reverse engineering framework; since the inception (a fork of radare2).

## Console Programs Not Supporting Truecolor

- [mutt](http://mutt.org/) (email client) - http://dev.mutt.org/trac/ticket/3674
- [neomutt](https://github.com/neomutt/neomutt) (email client) - [issue #58](https://github.com/neomutt/neomutt/issues/85)
- [termbox](https://github.com/nsf/termbox) library - [issue #37](https://github.com/nsf/termbox/issues/37) (there is a fork [termbox_next](https://github.com/cylgom/termbox_next) with the support
- [mcabber](https://mcabber.com/) (jabber client) - [issue #126](https://bitbucket.org/McKael/mcabber-crew/issue/126/support-for-true-color-16-millions-colors)
- [tig](https://github.com/jonas/tig) (git TUI) - [issue #227](https://github.com/jonas/tig/issues/227)
- [cmus](https://github.com/cmus/cmus) (music player) - [issue #799](https://github.com/cmus/cmus/issues/799)
- [weechat](https://github.com/weechat/weechat) (chat client) - [issue #1364](https://github.com/weechat/weechat/issues/1364)
- [scim](https://github.com/andmarti1424/sc-im) (spreadsheet program) - [issue #306](https://github.com/andmarti1424/sc-im/issues/306)
- [gui.cs](https://github.com/migueldeicaza/gui.cs) Terminal UI toolkit for .NET (curses-like) - [issue #48](https://github.com/migueldeicaza/gui.cs/issues/48)
