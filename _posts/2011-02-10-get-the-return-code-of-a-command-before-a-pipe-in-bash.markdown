---
author: keymon
comments: true
date: 2011-02-10 17:23:14+00:00
layout: post
slug: get-the-return-code-of-a-command-before-a-pipe-in-bash
title: Get the return code of a command before a pipe in Bash
wordpress_id: 253
---

Today I discovered a great functionality in bash:

hrivas@ADWMU001:~> false | cat ; echo $? ${PIPESTATUS[0]}
0 1

Great!
