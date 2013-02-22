---
layout: post
title: "On Editors"
date: 2013-02-22 14:17
comments: true
categories: 
---

"What's your favorite editor?" This question is almost bound to start a fight amongst even the most amiable of coders. *Nothing* boils programmer blood as much as an argument over dev tools. Of course, that never stops anyone from asking, and I've frequently debated the virtues of all sorts of programming tools, including of course the one true editor, Emacs.

Now, before you write me off as just another of those wacko Emacs users, hear me out. As much as I love Emacs, I firmly believe that every need deserves a careful look at which tool is best. For example, I am currently writing this post with a new tool I'm trying out, IA Writer, rather than my usual Emacs. I'll give a run-down of a few of the tools I use, and the needs they fulfill, and hopefully this will help you out in your journey to find the best tools for the job.

First, a bit of background: I've used a multitude of editors and environments, from QBasic in Windows 3.1 to Visual Studio. I've ran Windows, Linux, and OS X as my daily work OS at various times. Now with that said, I don't claim to be un-biased regarding programming tools. I am very partial to open source software, and will always choose the open source alternative if it is good enough for my needs.

What exactly are my needs? I write code on a daily basis, in a variety of languages. Lately, for example, I've needed to write C/C++, Java, Python, JavaScript, Latex, shell scripts, C#, PHP, and scrape together some HTML and CSS. My preference would naturally be to learn one tool that I can use for all these languages. I want to work as quickly and efficiently as possible, which means I need very useable tools. For me usability requires a simple, clean, keyboard-driven interface. I prefer to touch my mouse as little as possible while working, since the mouse disturbs my flow of work and is infinitely slower than a few quick key presses. Finally, for bonus points, I'd like my tools to be cross-platform since I switch back and forth on various machines and would like a somewhat consistent interface. So, this adds up to lots of languages, keyboard-friendly, and hopefully cross-platform.

These requirements leave me few choices for a primary editor: Kate (and similar basic text editors), Emacs, Vim, Eclipse, Netbeans, and Visual Studio. Visual Studio immediately fails my requirements, since it only excels for a small subset of the languages I use. However, I must admit that VS is easily the best tool I've found if you are primarily working in C# (sorry Mono…). I'm not a big fan of heavy IDE's such as Eclipse and Netbeans, since they tend to be a bit slow for me. In addition, both tend to focus primarily on Java, which I use as rarely as possible out of personal preference. This leaves a choice of either a basic text editor such as Kate or Notepad++, or something more complex, such as Emacs or Vim. I used Kate for a long time as my primary editor, but I never looked back after eventually switching to Emacs. I also used Vim for about two years, but found that my brain didn't work well with modal editing. Additionally, I was annoyed by the necessity to heavily customize Vim to make it useable for the way I work. I won't dwell too much on comparing Emacs to other editors, however I hope you can make your own comparison from the features I highlight.

## Starting out with Emacs

Now I've heard many complaints about the usability of Emacs, but most of this seems to stem from the complexity of shortcuts. First of all, you *must* use Caps Lock as your Control key to make any sense of Emacs. There's just no way around it, since Control is so crucial and yet so far away on modern keyboard layouts. Second, most of the important keybinds fit on the front and back of a standard letter size paper. Print out the Emacs quick reference, stick it next to your keyboard, and you'll quickly learn all the everyday shortcuts.

Unless otherwise noted, all code snippets should be placed in your `~/.emacs` file. For installing packages I try to use [ELPA](http://emacswiki.org/emacs/ELPA). Starting in the latest version (24), Emacs *finally* has built-in package installation support. This makes installing packages a breeze: `meta-x package-list-packages`, move the cursor to a package you want, hit `i` to mark for installation, then `x` to install. Done! ELPA can keep your packages up-to-date, and has a wide selection of common packages. I use the following config for access to all the usual repositories:
{% codeblock .emacs lang:cl %}
(setq package-archives '(("gnu" . "http://elpa.gnu.org/packages/")
                         ("marmalade" . "http://marmalade-repo.org/packages/")
                         ("melpa" . "http://melpa.milkbox.net/packages/")))
{% endcodeblock %}

## How I use Emacs
To give you a feel for how powerful Emacs is, I'll describe a few of my favorite tricks and features, such as navigation, LaTeX integration, and gdb integration.

### Favorite Tricks
I have a few favorite Emacs tricks and commands that I find terribly useful for what I need, and you might too.

I absolutely love [org-mode](http://orgmode.org/), but I think I would need to take an entire blog post to really do it justice. For now, I shall simply say that it is the simplest, greatest, all-around note-taking and organization software around. I know people who use Emacs solely for org-mode. It's that good.

[Hexl-mode](http://www.gnu.org/software/emacs/manual/html_node/emacs/Editing-Binary-Files.html) is a fairly nice builtin, although not very feature-complete, hex-viewer. It works great for quickly viewing some binary data, but isn't quite enough if you plan to do a lot of hex hacking.

While not really a trick, I love the way that Emacs handles indentation. Emacs always "does the right thing" after you set up your indent style, such as with this config:
{% codeblock .emacs lang:cl %}
(setq c-default-style "k&r"
      c-basic-offset 2)

(setq default-tab-width 2)
{% endcodeblock %}
Hitting tab always indents the current line to the right position, regardless of where it started. This is great for making all of your indentation consistently perfect. If you need to re-indent a section, just select the section and hit `meta-\` to fix all the indentation in that section. While I've found most modes do a great job of auto-indentation, unfortunately the js2-mode and HTML modes tend to fall short. However, on the whole, I much prefer this single tab to indent style rather than forcing the programmer to think about how many times to hit tab.

### How to get around
I like having a lot of buffers (files) open at once. In fact I'm guilty of this in general -- my browser usually has at least 150 tabs open. Fortunately, Emacs makes this super easy. First, I need it to remember all my open files between sessions:
{% codeblock .emacs lang:cl %}
(desktop-save-mode 1)
{% endcodeblock %}

For a long time I simply used `ctrl-x b` to switch buffers. By default, this buffer switch does a few helpful things. The default buffer is the most recently used buffer, which is helpful for flipping back and forth between a few files, and older buffers are quickly accessibly via the up arrow. However, I found that [IDO](http://emacswiki.org/emacs/InteractivelyDoThings) makes switching buffers and opening files *much* easier. IDO is an unobtrusive plugin which completes buffer and filenames as you type, generally in a natural manner. IDO is built in to recent versions of Emacs and is enabled with:
{% codeblock .emacs lang:cl %}
(require 'ido)
(ido-mode t)
{% endcodeblock %}
I tend to use the following options for extra goodness:
{% codeblock .emacs lang:cl %}
(ido-everywhere t)
(setq ido-enable-flex-matching t)
(setq ido-create-new-buffer 'always)
(setq ido-enable-tramp-completion nil)
(setq ido-confirm-unique-completion nil)
{% endcodeblock %}


### Modes

As a grad student now, I tend to write a lot of stuff in LaTeX, so I need a good editing environment for that. Along with almost every other Emacs user, my mode of choice is [AUCTeX](http://www.gnu.org/software/auctex/), but I augment this with a few additions. First of all, I use the following settings for AUCTeX:
{% codeblock .emacs lang:cl %}
(setq TeX-PDF-mode t)

(add-hook 'LaTeX-mode-hook 'visual-line-mode)
(add-hook 'LaTeX-mode-hook 'speck-mode)
(add-hook 'LaTeX-mode-hook 'LaTeX-math-mode)
(add-hook 'LaTeX-mode-hook 'writegood-mode)
(add-hook 'TeX-mode-hook 'zotelo-minor-mode)

(add-hook 'LaTeX-mode-hook 'turn-on-reftex)
(setq reftex-plug-into-AUCTeX t)
{% endcodeblock %}

These settings set the default output to PDF, turn on visual line mode (line navigation respects soft-wraps), turn on [speck](http://www.emacswiki.org/SpeckMode) mode for spell checking (an alternative to [flyspell](http://www.emacswiki.org/emacs/FlySpell), which is also great), turn on math mode for LaTeX since I write a lot of math, and turn on reftex and [integration with Zotero](https://github.com/vitoshka/zotelo), my reference manager. I won't go into a lot of detail on these, since the manuals for each tool tend to be fairly good.

I have found that [writegood-mode](https://github.com/bnbeckwith/writegood-mode) is great for keeping my writing clean and straightforward when I'm writing papers. It simply marks common errors such as repeated words, as well as uses of the passive voice, and weak or overused words. Try it out, and see what you think!

I use the usual modes for editing various sources, such as js2-mode, python-mode, etc. for syntax highlighting and other goodness. These modes are all available from ELPA, and a quick google or check of the Emacs wiki should point you in the right direction.

### GDB
The gdb integration in Emacs is so useful it deserves a section of its own. To fire up gdb in emacs, simply use `meta-x gdb`. This will prompt you for a command to run gdb with, defaulting to the current executable if your working directory is simple enough to figure this out. You can use any of the usual gdb options here, but make sure that you keep the `-i=mi` bit at the beginning -- this allows Emacs to integrate gdb nicely into its interface.

The best way to use gdb, assuming you have enough screen space, is the many-windows mode. Simply hit `meta-x gdb-many-windows` after starting a gdb session, and Voilà, you have a gdb prompt, current source file, local vars, breakpoints, etc. Breakpoints can be set in open source files simply by navigating to the line you want to break on and hitting `ctrl-x SPACE`. Breakpoints are disabled simply by finding the breakpoint in the breakpoints buffer and hitting `D` on it. Hitting spacebar on a breakpoint disables it. You can also do lots more fun things with gdb, such as open buffers for live memory dumps or disassembly views. Check out the manual for more details.

## Now go forth and conquer

I keep my configs in git, which allows me to easily branch for any new machine or platform I'm running on. Check out my exact configs on [github](http://github.com/rinon/configs) and let me know if you have any suggestions or comments!

I hope this post is useful to some, and that others would have great ideas that I can incorporate into my editing workflow. Please comment or contact me (links in the header) if you have any suggestions or comments!