---
title: 使其他设备通过网络访问wsl
date: 2022-12-27 12:24:18
tags:
---

# 使其他设备通过网络访问wsl

## 原理

1. 获取wsl的ip
2. 添加防火墙规则
3. 添加端口转发规则

## 脚本

```powershell
$remoteport = bash.exe -c "ifconfig eth0 | grep 'inet '"
$found = $remoteport -match '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}';

if( $found ){
  $remoteport = $matches[0];
} else{
  echo "The Script Exited, the ip address of WSL 2 cannot be found";
  exit;
}

#[Ports]

#All the ports you want to forward separated by coma
$wslports=@(22,8001,8002,10000,3000,5000);
$winports=@(222,8001,8002,10000,3000,5000);


#[Static ip]
#You can change the addr to your ip config to listen to a specific address
$addr='0.0.0.0';
$ports_a = $winports -join ",";


#Remove Firewall Exception Rules
iex "Remove-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' ";

#adding Exception Rules for inbound and outbound Rules
iex "New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Outbound -LocalPort $ports_a -Action Allow -Protocol TCP";
iex "New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Inbound -LocalPort $ports_a -Action Allow -Protocol TCP";

for( $i = 0; $i -lt $wslports.length; $i++ ){
  $wslport = $wslports[$i];
  $winport = $winports[$i];
  iex "netsh interface portproxy delete v4tov4 listenport=$winport listenaddress=$addr";
  iex "netsh interface portproxy add v4tov4 listenport=$winport listenaddress=$addr connectport=$wslport connectaddress=$remoteport";
}

echo "Success";
```

## 参考

https://github.com/microsoft/WSL/issues/4150#issuecomment-504209723