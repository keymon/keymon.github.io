---
author: keymon
comments: true
date: 2010-06-08 10:03:56+00:00
layout: post
slug: extending-puppet-for-aix-manage-nfs-mountpoints
title: 'Extending puppet for AIX: manage nfs mountpoints'
wordpress_id: 118
categories:
- aix
- puppet
- script
- sysadmin
tags:
- aix
- automatization
- chnfsmnt
- configuration
- mknfsmnt
- mountpoint
- nfs
- puppet
- rmnfsmnt
- sysadmin
---

I am playing arourd with [puppet](http://www.puppetlabs.com/), a configuration management software.

We have both AIX and Linux systems, but I find puppet a little bit inmature for AIX.

Anyway, I think that it will be easy implement providers and recipes using the Exec providers on AIX. AIX has a consistent set of commands, and almost everything can be configured from command line. Most of commands have similar options, syntax and ouput.

Normally, all OS configuration items (users, groups, mountpoints) have a set of commands: ls, ch and mk. Most of then are based on attributes that can be changed/set and output of ls* commands usually can be printed in colons (-c option).

<!-- more -->
For instance, here you have a definition that will configure a nfs mountpoint on aix, using `mknfsmnt`, `chnfsmnt` and `rmnfsmnt`. Note: I am a completely newbie in puppet, probably it can be done better.

[sourcecode language="python"]
# Define to implement AIX nfs mountpoints. It is based on commands lsnfsmnt and mknfsmnt.
# It will:
#  * Check if the mountpoint is defined.
#  * If is defined, check the options.
#  * If options change, remove it and create it (it will umount if needed)
#
# ensure:
#  * present: Create/Change it in configuration and running mounts
#  * absent: Create/Change it in configuration and running mounts
#  * configured: Change only configuration
#  * mounted: Do not change configuration, just mount it.
#  * umounted: Do not change configuration, just umount it.
#  * unconfigured: Do not umount it, remove configuration
define aix_nfsmnt(
		 $ensure=present, # Valid values:
		 $force=false,

         $mountpoint="",
         $remotepath,
         $nodename,

         $type="nfs",
         $readtype="rw", # rw|ro
         $when="", #  bg|fg
         $softhard="", # soft | hard
         $secure="", # more secure true|false (-s|-n)
         $intr="", # Interrupt true - false
         $retry="",
         $rsize="",
         $wsize="",
         $timeo="",
         $port="",
         $acregmin="",
         $acregmax="",
         $acdirmin="",
         $acdirmax="",
         $actimeo="",
         $biods="",
         $suid="", # set uid true or false
         $dev="", # devs true or false
         $shortdev="", # sort dev true or false
         $auto="", # Auto value
         $vers="",
         $proto="any",
         $llock="", # file locking true or false (Not documented) -L|-l
         $acl="", #acl true|false -J|-j
         $grpid="", #inherit group false|true -g|-G
		 $posix="", # v2 posix  pathconf
	     $retrans="") {

	# Get the default name as mountpoint if no mountpoint given
	$_mountpoint 	= $mountpoint? 	{ "" => $name,	default => $mountpoint,	}

	# Possible options
	$_type 		 	= $type? 		{ "" => "",	default => "-m $type"}
	$_readtype 		= $readtype? 	{ "" => "",	default => "-t $readtype" }
	$_when 			= $when? 		{ "" => "",	default => "-w $when" }
	$_softhard 		= $softhard? 	{ "soft" => "-S", "hard" => "-H", default => "", }
	#$_secure 		= $secure? 		{ true => "-s", false => "-n", default => "", }
	$_secure 		= $secure? 		{ "" => "", default => "-M $secure", }
	$_intr 			= $intr? 		{ false => "-e", true => "-E",	default => "", }
    $_retry 		= $retry? 		{ "" => "", default => "-r $retry", }
    $_rsize 		= $rsize? 		{ "" => "", default => "-b $rsize", }
    $_wsize 		= $wsize? 		{ "" => "", default => "-c $wsize", }
    $_timeo 		= $timeo? 		{ "" => "", default => "-o $timeo", }
    $_port 			= $port? 		{ "" => "", default => "-P $port", }
    $_acregmin 		= $acregmin? 	{ "" => "", default => "-u $acregmin", }
	$_acregmax 		= $acregmax? 	{ "" => "", default => "-U $acregmax", }
    $_acdirmin 		= $acdirmin?    { "" => "", default => "-v $acdirmin", }
    $_acdirmax 		= $acdirmax? 	{ "" => "", default => "-V $acdirmax", }
    $_actimeo 		= $actimeo? 	{ "" => "", default => "-o $actimeo", }
    $_biods 		= $biods? 		{ "" => "", default => "-p $biods", }
	$_suid 			= $suid? 		{ false => "-y", true => "-Y", default => "", }
	$_dev 		    = $dev? 		{ false => "-z", true => "-Z",	default => "", }
	$_shortdev 		= $shortdev? 	{ false => "-x", true => "-X", default => "-X", }
	$_auto 			= $auto? 		{ false => "-a", true => "-A", default => "", }
    $_vers 			= $vers? 		{ "" => "", default => "-K $vers", }
	$_proto 		= $proto? 		{ "" => "", default => "-k $proto", }
	$_llock 		= $llock? 		{ false => "-l", true => "-L", default => "", }
	$_acl 			= $acl?			{ false => "-j", true => "-J", default => "", }
	$_grpid 		= $grpid?		{ false => "-g", true => "-G", default => "", }
	$_posix 		= $posix?		{ false => "-q", true => "-Q", default => "-q", }
	$_retrans       = $retrans? 	{ "" => "", default => "-R $retrans", }

	$_force_umount	= $force?		{ true => "-f", default => "", }

	# Which action perform, and if it is persistent or not
	case "$ensure" {
		present: {
			$mountaction="-B"
			$domount=true
			$docreation=true
		}
		absent: {
			$mountaction="-B"
			$domount=true
			$docreation=false
		}
		mounted: {
			$mountaction="-I"
			$domount=true
			$docreation=true
		}
		umounted: {
			$mountaction="-I"
			$domount=true
			$docreation=false
		}
		configured: {
			$mountaction="-N"
			$domount=false
			$docreation=true
		}
		unconfigured:{
			$mountaction="-N"
			$domount=false
			$docreation=false
		}
		default: { fail("Unkown ensure value '$ensure'") }
	}

	$expected_lsnfsmnt_options = "$_mountpoint:$remotepath:$nodename:$type:$readtype:$when:$_softhard:$secure:$_intr:$retry:$rsize:$wsize:$timeo:$port:$acregmin:$acregmax:$acdirmin:$acdirmax:$actimeo:$biods:$_suid:$_dev:$_shortdev:$_auto:$vers:$proto:$_llock:$_acl:$_grpid:$_posix:$retrans"

	$nfsmnt_args="$mountaction -f $_mountpoint -d $remotepath -h $nodename $_type $_readtype $_when $_softhard $_secure $_intr $_retry $_rsize $_wsize $_timeo $_port $_acregmin $_acregmax $_acdirmin $_acdirmax $_actimeo $_biods $_suid $_dev $_shortdev $_auto $_vers $_proto $_llock $_acl $_grpid $_posix $_retrans"

	#notify{"The value 'expected_lsnfsmnt_options_cmd' is: ${expected_lsnfsmnt_options}": }
	#notify{"The value 'nfsmnt_args' is: ${nfsmnt_args}": }

	# Create mountpoint if needed
	file { $_mountpoint:
		ensure => directory,
	}

	# Commands to umount the fs before execute the rmnfsmnt.
	if $domount {
		$pre_rmnfsmnt_cmd="/usr/sbin/umount $_force_umount $_mountpoint"
		$postfail_rmnfsmnt_cmd="(RET=\$?; /usr/sbin/mount $_mountpoint; exit \$RET)"
	} else {
		$pre_rmnfsmnt_cmd="/usr/bin/true"
		$postfail_rmnfsmnt_cmd="(RET=\$?; exit \$RET)"
	}

	if $docreation {
		# Remove the old definition if options change. To check it, it will list the options in table mode.
		# Umount before remove if needed.
		exec{"rmnfsmnt-$title":
			command => "$pre_rmnfsmnt_cmd && \
						/usr/sbin/rmnfsmnt -f $_mountpoint || \
						$postfail_rmnfsmnt_cmd",
			cwd => "/",
			onlyif => "/usr/sbin/lsnfsmnt -c $_mountpoint",
			unless => "/usr/sbin/lsnfsmnt -c $_mountpoint | /usr/bin/grep -e '^$expected_lsnfsmnt_options\$'",
			}

		# Create the definition. It will remove previous if needed. If previous definition has
		# same options, rmnfsmnt will not be executed, and the unless attribute will prevent this
		# command to execute.
		exec{"mknfsmnt-$title":
			require => Exec["rmnfsmnt-$title"],
			command => "/usr/sbin/mknfsmnt $nfsmnt_args",
			cwd => "/",
			unless => "/usr/sbin/lsnfsmnt -c $_mountpoint | /usr/bin/grep -e '$_mountpoint:$remotepath:$nodename'",
			}
	} else {
		exec{"rmnfsmnt-$title":
			command => "$pre_rmnfsmnt_cmd && \
						/usr/sbin/rmnfsmnt -f $_mountpoint || \
						$postfail_rmnfsmnt_cmd",
			cwd => "/",
			onlyif => "/usr/sbin/lsnfsmnt -c $_mountpoint",
			}
	}
}
[/sourcecode]
