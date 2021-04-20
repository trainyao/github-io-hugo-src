---
author: "trainyao"
date: 2020-03-25
linktitle: kubectl custom tool snippet
title: kubectl custom tool snippet
categories: ["其他"]
weight: 10
---

```bash
#!/bin/bash

do=$1
pod=$2
t=
if [[ $do == "e" ]]; then
	do=exec
	t="-it bash"
fi
if [[ $do == "l" ]]; then
	do=logs
fi
if [[ $do == "d" ]]; then
	do="delete pods"
fi
if [[ $do == "de" ]]; then
	do="get pods"
	t="-o yaml"
fi
echo $do

podname=`kubectl get pods | grep ${pod} | grep Running | head -n 1 | awk '{print $1}'`
echo $podname

echo "kubectl ${do} ${podname} ${t} $3 $4 $5 $6"

kubectl ${do} ${podname} ${t} $3 $4 $5 $6
```
