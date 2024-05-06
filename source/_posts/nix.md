---
title: nix笔记
date: 2024-05-06 17:16:00
tags:
---

# nix笔记

## root

- /nix/var/nix/profiles/default
- /nix/var/nix/profiles/per-user/username/profile 不配置use-xdg-base-directories为true时
- /home/username/.local/state/nix/profiles/profile 配置了use-xdg-base-directories为true时

根据NIX_PROFILES环境变量来获取所有环境

## 回收

使用这个命令可以拿到当前系统依赖的root link

``` shell
nix-store --gc --print-roots
```

使用这个可以查看这个link下有哪些程序

``` shell
nix-tree /nix/var/nix/profiles/default
```

使用这个可以查看关联图与大小

``` shell
nix-du -n=50 | dot -Tsvg > store.svg
nix-du -s=500MB | dot -Tsvg > store.svg
nix-du -n=50 --root /run/current-system/sw/ | dot -Tsvg > store.svg
```

使用这个可以查看世代与删除世代

``` shell
nix-env --profile /nix/var/nix/profiles/default --list-generations
nix-env --profile /nix/var/nix/profiles/default --delete-generations old

````

