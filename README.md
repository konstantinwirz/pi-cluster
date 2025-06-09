# Raspberry PI Cluster Setup
[![ansible-lint](https://github.com/konstantinwirz/pi-cluster/actions/workflows/ansible-lint.yaml/badge.svg)](https://github.com/konstantinwirz/pi-cluster/actions/workflows/ansible-lint.yaml)

## Requirements
 - ansible (tested with 2.16.3)
 - more than 1 raspberry pi (tested with raspberry pi 5)

 ## Get kubeconfig

 ```shell
 scp pi1:/etc/rancher/k3s/k3s.yaml k3s.yaml
# Update the server address in k3s.yaml
sed -i '' 's/127.0.0.1/192.168.1.131/g' k3s.yaml
```

## PiHole

- [Helm Chart](https://github.com/MoJo2600/pihole-kubernetes/tree/main)
- [The Big Blocklist Collection](https://firebog.net)