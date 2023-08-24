---
title: acme_build_cert.md
date: 2021-11-01 11:32:00
tags:
---

# acme.sh build certification

## env

* Ubuntu 20.04 LTS
* domain name -> wincher.cn

## operations

* install

```bash
$ curl https://get.acme.sh | sh
[Mon Nov  1 10:52:26 CST 2021] Installing from online archive.
[Mon Nov  1 10:52:26 CST 2021] Downloading https://github.com/acmesh-official/acme.sh/archive/master.tar.gz
[Mon Nov  1 10:52:28 CST 2021] Extracting master.tar.gz
[Mon Nov  1 10:52:28 CST 2021] It is recommended to install socat first.
[Mon Nov  1 10:52:28 CST 2021] We use socat for standalone server if you use standalone mode.
[Mon Nov  1 10:52:28 CST 2021] If you don't use standalone mode, just ignore this warning.
[Mon Nov  1 10:52:28 CST 2021] Installing to /home/wincher/.acme.sh
[Mon Nov  1 10:52:28 CST 2021] Installed to /home/wincher/.acme.sh/acme.sh
[Mon Nov  1 10:52:28 CST 2021] Installing alias to '/home/wincher/.zshrc'
[Mon Nov  1 10:52:28 CST 2021] OK, Close and reopen your terminal to start using acme.sh
[Mon Nov  1 10:52:28 CST 2021] Installing cron job
no crontab for wincher
no crontab for wincher
[Mon Nov  1 10:52:28 CST 2021] Good, bash is found, so change the shebang to use bash as preferred.
[Mon Nov  1 10:52:29 CST 2021] OK
[Mon Nov  1 10:52:29 CST 2021] Install success!

```

```bash
$ acme.sh --issue --domain wincher.cn --standalone
[Mon Nov  1 11:17:49 CST 2021] Using CA: https://acme.zerossl.com/v2/DV90
[Mon Nov  1 11:17:49 CST 2021] Please install socat tools first.
[Mon Nov  1 11:17:49 CST 2021] _on_before_issue.
```

* go install [socat](http://www.dest-unreach.org/socat/)

  ```bash
  $ sudo apt install -y socat
  ```

* issue again

  ```bash
  $ acme.sh --issue --standalone -d wincher.cn
  [Mon Nov  1 12:39:01 CST 2021] Using CA: https://acme.zerossl.com/v2/DV90
  [Mon Nov  1 12:39:01 CST 2021] Standalone mode.
  [Mon Nov  1 12:39:01 CST 2021] LISTEN    0         511                      *:80                     *:*
  [Mon Nov  1 12:39:01 CST 2021] tcp port 80 is already used by
  [Mon Nov  1 12:39:01 CST 2021] Please stop it first
  [Mon Nov  1 12:39:01 CST 2021] _on_before_issue.
  ```

  we can see that this operation will use your sever's port:80, then and my server just run a application that use 80 port, so I stop it.

* issue again

  ```
  ```

  

