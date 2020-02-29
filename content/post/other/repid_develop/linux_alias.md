---
author: "trainyao"
date: 2018-02-09 09:44:45
linktitle: 类unix系统提示效率的alias
title: 类unix系统提示效率的alias
categories: ["其他"]
weight: 10
---

``` shell
# git
alias gs='git status '
alias ga='git add '
# usage: gc 'comment info' 
alias gc='git commit -m '
alias gd='git diff  '
alias gsh='git show '
alias grm='git rm '
alias gck='git checkout '
alias gps='git push origin '
alias gpl='git pull origin '
alias gpl='git branch | grep \* | cut -d " " -f 2 | xargs git pull origin '
alias grs='git reset '
alias gl='git log '
alias gb='git branch '
alias gmg='git merge '
alias gst='git stash '
alias gf='git fetch'
alias grollback="gl | grep commit | head -n 1 | awk '{print \$2}' | xargs -I {} git reset --hard {}"
alias u='gck online
gb -D develop
gb -D master
git fetch
gck develop
gck master
'

# other
alias ll='ls -al'
alias ip='ifconfig | grep 172'

# docker
alias de='docker exec -it '
alias dp='docker ps'
alias dpa='docker ps -a'
alias di='docker images'
alias drmi='docker rmi '
alias drm='docker rm '
alias dr='docker run '
alias dk='docker kill '
```
