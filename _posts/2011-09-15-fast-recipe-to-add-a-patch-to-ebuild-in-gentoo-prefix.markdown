---
author: keymon
comments: true
date: 2011-09-15 12:56:29+00:00
layout: post
slug: fast-recipe-to-add-a-patch-to-ebuild-in-gentoo-prefix
title: Fast recipe to add a patch to ebuild in gentoo-prefix
wordpress_id: 258
categories:
- fast-tip
- gentoo
- sysadmin
- Technical
---

In this case I needed to patch lftp.

First we set the configuration to support the overlay path (This is done once)

`
export PORTDIR_OVERLAY="$EPREFIX/usr/local/portage"
cat <<EOF >>$EPREFIX/etc/make.conf
# Overlay
PORTDIR_OVERLAY="$PORTDIR_OVERLAY"
EOF
`

And then, for any package, we just have to copy the ebuild and its files, and add the new patch (copying the file and updating the ebuild):

`
# To create a overlay version of any package, just change this variables
pkg=net-ftp/lftp
pkgvers=lftp-4.3.1

# Copy the ebuild
mkdir -p $PORTDIR_OVERLAY/$pkg
cp $EPREFIX/usr/portage/$pkg/$pkgvers.ebuild  $PORTDIR_OVERLAY/$pkg
cp -R $EPREFIX/usr/portage/$pkg/files   $PORTDIR_OVERLAY/$pkg/files

# Do any change.
# e.p. a Simple modification: add patches and add them to the ebuild:
#  cp lftp-solaris-2.10-socket.patch $PORTDIR_OVERLAY/$pkg/files/lftp-solaris-2.10-socket.patch
#  joe $EPREFIX/usr/portage/$pkg/$pkgvers.ebuild
#    +> Add to src_prepare(): epatch "${FILESDIR}/${PN}-solaris-2.10-socket.patch"

# Sign the ebuild
ebuild $PORTDIR_OVERLAY/$pkg/$pkgvers.ebuild digest
`

