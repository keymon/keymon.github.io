---
author: keymon
comments: true
date: 2012-01-15 03:16:12+00:00
layout: post
slug: syntax-highlighting-in-less-bashrc-snippet
title: Syntax highlighting in less. Bashrc snippet
wordpress_id: 275
categories:
- fast-tip
- script
- trick
---

Based to this post, http://linux-tips.org/article/78/syntax-highlighting-in-less my fast tip to allow syntax highlight in less:

[sourcecode language="bash"]
cat <<EOF >> ~/.bash_profile

# Syntax Highlight for less
#
# Check if source highlite is intalled http://www.gnu.org/software/src-highlite/
# Set SRC_HILITE_LESSPIPE for custom location
# 
# To install: 
#   sudo yum install source-highlight
#
SRC_HILITE_LESSPIPE=${SRC_HILITE_LESSPIPE:-$(which src-hilite-lesspipe.sh 2> /dev/null)}
if [ -x "$SRC_HILITE_LESSPIPE" ]; then
	export LESSOPEN="| $SRC_HILITE_LESSPIPE  %s"
	export LESS="${LESS/ -R/}  -R" # Set LESS option in raw mode (avoid repeating it)
fi
EOF
[/sourcecode]
