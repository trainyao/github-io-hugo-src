---
author: "trainyao"
date: 2021-09-14
linktitle: tmux 配置
title: tmux 配置
categories: ["其他"]
weight: 10
---

``` text
bind -r k select-pane -U # 绑定k为↑
bind -r j select-pane -D # 绑定j为↓
bind -r h select-pane -L # 绑定h为←
bind -r l select-pane -R # 绑定l为→


bind -r ^k resizep -U 5 # 绑定Ctrl+k为往↑调整面板边缘10个单元格
bind -r ^j resizep -D 5 # 绑定Ctrl+j为往↓调整面板边缘10个单元格
bind -r ^h resizep -L 5 # 绑定Ctrl+h为往←调整面板边缘10个单元格
bind -r ^l resizep -R 5 # 绑定Ctrl+l为往→调整面板边缘10个单元格

run-shell ~/.tmux/tmux-resurrect/resurrect.tmux

bind -r V copy-mode # 绑定Ctrl+l为往→调整面板边缘10个单元格

bind-key -T copy-mode v   send -X begin-selection  # default is <space>
bind-key -T copy-mode V   send -X select-line
bind-key -T copy-mode C-v send -X rectangle-toggle  # default is C-v, or R in copy-mode (non-vi)
bind-key -T copy-mode y   send -X copy-pipe-and-cancel 'xclip -selection clipboard -in'
bind p paste-buffer  # default ]
```
