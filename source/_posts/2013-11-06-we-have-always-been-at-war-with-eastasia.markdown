---
layout: post
title: "We have always been at war with eastasia"
date: 2013-11-06 23:12
comments: true
categories: 
---

Rewriting history when doing an svn to git migration is always fun. Especially when you start rewriting the entire contents of files.

Take this ditty for instance:

>    git filter-branch -f --tree-filter 'for j in $(for i in $(find . -name \*.pp | grep -v vendor | grep -v tmp); do cat $i |perl -ne"/(\s+)\S/&&\$e{length(\$1)}++;END {exit 0 if \$e{2}; exit 2}"|| echo $i; done); do cat $j | perl -ne"/^(\s+)/;\$s=length(\$1)/2;\$s=\" \"x\$s;s/^\s+/\$s/;print">"$j.new" && mv "$j.new" $j; done;find . -name \*.pp | grep -v vendor | grep -v tmp | xargs puppet-lint --fix >/dev/null ||true' HEAD

This switches my puppet codebase to the canonical 2 space (rather than 4 space) tabs and fixes lint/quoting issues - all the way back through history.

I, for one, welcome our newold code layout overlords.

