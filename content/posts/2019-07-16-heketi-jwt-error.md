+++
title= "Heketi JWT token expired error"
date= "2019-07-16"
tags= ["Heketi", "Gluster"]
+++

Recently I encountered an `JWT token expired` error on heketi pod in Openshift cluster.

```bash
[jwt] ERROR 2019/07/16 19:17:14 heketi/middleware/jwt.go:66:middleware.(*HeketiJwtClaims).Valid: exp validation failed: Token is expired by 1h48m59s
```

After a lot of google search, I synchronized clocks across the pod running heketi and the masters, which solved the issue

```bash
$ ntpdate -q 0.rhel.pool.ntp.org; systemctl restart ntpd
```
