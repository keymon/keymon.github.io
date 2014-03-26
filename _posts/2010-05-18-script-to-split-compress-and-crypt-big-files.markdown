---
author: keymon
comments: true
date: 2010-05-18 10:25:15+00:00
layout: post
slug: script-to-split-compress-and-crypt-big-files
title: Script to split, compress and crypt big files
wordpress_id: 75
categories:
- fast-tip
- script
- sysadmin
tags:
- bash
- compress
- gpg
- gzip
- script
- split
- sysadmin
- tip
- trick
- upload
---

I am playing with [git ](http://git-scm.com/)and [github](https://github.com/). I think that I will [upload to this repository (http://github.com/keymon/snippets](http://github.com/keymon/snippets)) all my small scripts and stuff.

Here goes one script to split, compress and asymetrically crypt big files. It is really usefull to upload or send big files to support.

It uses dd to split the files, gzip to compress them and gpg to optionally crypt. It will also uncompress or check the stripes.

<!-- more -->Here you are the script:



	
  * Download raw: [http://github.com/keymon/snippets/raw/master/scripts/misc/file-splitter.sh](http://github.com/keymon/snippets/raw/master/scripts/misc/file-splitter.sh)

	
  * View source: [http://github.com/keymon/snippets/blob/master/scripts/misc/file-splitter.sh](http://github.com/keymon/snippets/blob/master/scripts/misc/file-splitter.sh)


The usage help:

    
    This command splits and compress (or uncompress) using gzip big files.
    It can crypt files symmetrically with gpg.
    It can be interrupted, and it will check the last split or all of them.
    
    Usage:
     file-splitter.sh [-c|-u] [-e <password>] [-s <split size in MB>] <filename> [filename ...]
    
     filename:
     - List of files to split and compress (and optionally crypt)
     - List of split files to check/uncompress
     - List of destination files whose splits will be checked/uncompressed
    
     -c: Compress (default action).
     -u: Uncompres the files,
     -v: Verify compressed files, but dot not perform any action.
     -e: Encrypt/Decrpt with gpg using given password
     -s: Set different split size (default 100MB)
     -t: Test the file integrity? This option is valid for compress,
     checking files already compressed, or to uncompress,
     checking files before uncompress.
     -f: Overwrite file if exists
    Example:
     file-splitter.sh -e "Gs4.2GPsa" -s 100 -t MUREX_20100131_01.dmp
    
    Known bugs:
     Do not use spaces os special characters in files.
    
