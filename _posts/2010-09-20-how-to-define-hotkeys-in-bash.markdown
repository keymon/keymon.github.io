---
author: keymon
comments: true
date: 2010-09-20 12:32:32+00:00
layout: post
slug: how-to-define-hotkeys-in-bash
title: How to define "hotkeys" in bash
wordpress_id: 215
categories:
- fast-tip
- linux/unix
- sysadmin
- trick
tags:
- bash
- linux
- scripting
- trick
---

For instance, I will define a hotkey to get manual page of current command without execute it (ideal for F1).

First, you get the code of the "hotkey" you want to use by pressing "Ctrl+U+<hotkey>". For example:

* Ctrl+L: ^L
* Ctrl+J: ^J
* F1: ^[OP

This code may vary from terminal to terminal.

First you define an function, called single-man, to execute man of the first argument:

    
    single-man() { man $1; }


Then, you add a line like this one in your .inputrc:

    
     "^[OP" "\C-A\C-K single-man \C-Y\C-M\C-Y"


What the hell does this? Well, when "F1" is pressed, in will simulate the press of "Ctrl+A", that goes to the begining of the line, "Ctrl+K" that copies current line to clipboard, "Ctrl+Y" that pastes the clipboard, "Ctrl+M" that press Enter and Ctrl+Y that pastes the clipboard one more time.

I use this trick since several years ago.
