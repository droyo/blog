+++
date = "2014-04-26T18:18:11-05:00"
title = "Bulk-upload files to artifactory"
description = "Simple shell script to upload directories to Artifactory"
tags = ["artifactory"]
+++

I found myself needing to upload a large amount of RPM files to
artifactory. I don't have maven installed, and am not too familiar
with it, so I took a look at Artifactory's [REST API][0] documentation
and put together a small shell script to do what I need:

	#!/bin/sh
	# Upload directory to artifactory
	
	die() {
		printf %s\\n "$*" >&2
		exit ${rc:-1}
	}
	
	usage() {
		prog=`basename "$0"`
		rc=2 die "Usage: ${prog} [-s url] [-u user] [-p pass] srcdir dstdir"
	}
	
	askpass() {
		read -rsp "$1: " pass
		echo>/dev/tty
		printf '%s' "$pass"
	}
	
	# Remove leading and trailing slashes
	trim() {
		printf %s "$1" | sed 's,^/\+,,;s,/\+$,,'
	}
	
	upload() {
		src="$1"; shift
		dst=`trim "$1"`; shift
	
		status=`(cd "$src" && tar c .) | upload_tar "$url/$dst/.tar"`
		case "$status" in
		(200) return 0;;
		(403) die "403 Forbidden" ;;
		(401) die "401 Invalid credentials" ;;
		(*) die "Error code ${status}";;
		esac
	}
	
	upload_tar() {
		dst="$1"; shift
		
		auth="${user}:${pass}"
		curl -o /dev/null -# -u "$auth" -X PUT -w '%{http_code}' \
			-H 'X-Explode-Archive: true' \
			--data-binary @/dev/stdin "$dst"
	}
	
	user="$USER"
	url="https://artifactory.example.com"
	
	while getopts :u:p:s: arg
	do case "$arg" in
		(u) user="$OPTARG"           ;;
		(p) pass="$OPTARG"           ;;
		(s) url=`trim "$OPTARG"` ;;
		(*) usage                    ;;
	esac; done
	shift `expr $OPTIND - 1`; unset OPTIND
	
	[ "$pass" ] || pass=`askpass "Artifactory password for ${user}"`
	[ $# -eq 2 ] || usage
	
	upload "$@"

I believe most of the REST API in artifactory is restricted in the free version. However, if you use Artifactory pro, this script might save you a bit of time.

[0]: http://www.jfrog.com/confluence/display/RTF/Artifactory+REST+API
