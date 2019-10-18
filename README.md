# k3s-config

Configuration for managing my local [`k3s`] cluster.

## Setup

I have it running in an ArchLinux box, and my `pacman` wrapper of choice is
[`pikaur`]:

```shell
pikaur -S k3s-bin
```

It'll install a `systemd` unit file alongside `k3s`, and I've decided to
modify the pre-packaged `traefik` install, so we'll need to modify this.

```shell
sudo systemctl edit --full k3s.service
```

Modify the `ExecStart` option to include the `--no-deploy traefik` flag,
save, and close.

Then start `k3s`:

```shell
sudo systemctl start k3s.service
```

On first boot it will generate a kubeconfig for you. If you have other clusters
configured in `~/.kube/config` then you'll need to manually merge them. The
generated config is written in `/etc/rancher/k3s/k3s.yaml`.

Otherwise, you can use `sudo k3s kubectl` as a `kubectl` replacement.

Next, create local hosts file entries for the cluster by running the helper
script in `scripts`. You'll want to make a few modifications before running
it:

- Change the interface to whatever your primary card is that gets an IP from
your router
- Change the hostname to whatever you want
- Backup /etc/hosts to your `$HOME`, since there are no guardrails on this script

```shell
sudo ./scripts/update_k3s_host
```

Then install `traefik`:

```shell
kubectl apply -f services/traefik.yaml
```

If all goes well, you should be able to navigate to `traefik.gondolin.k3s`,
or whatever you changed the base hostname to, and see the `traefik` dashboard.

## Local Storage

There is a "host path provisioner" that uses the host machine's filesystem
to store data from PVC claims in the cluster. You can enable it by running:

```sh
kc apply -f services/storage-class.yaml
```

Note you'll have to update the YAML file to put in a host path that is writeable
on your system.

## Docker Registry

If you'd like to build and publish containers to a registry running in the
cluster set one up like so:

```sh
cfssl gencert -cinitca ca.json | cfssljson -bare ca
ccfssl gencert -ca=ca.pem -ca-key=ca-key.pem -profile=server -hostname=gondolin serverRequest.json | cfssljson -bare registry
```

Then create the certs in the cluster like so:

```sh
kc -n kube-system create secret tls registry-ingress-tls --cert=registry.pem --key=registry-key.pem
```

Then create the registry:

```sh
kc apply -f services/registry/registry.yaml
```

Note that this doesn't work currently.

## Running Locally Built Containers in k3s

Apparently you have to preload them into `k3s` 😐

https://rancher.com/docs/k3s/latest/en/running/#air-gap-support

So presumably the idea is that you `docker save` the thing you're working on,
pre-load it using this air-gap support shenanigans, then restart `k3s`. I
haven't gotten around to testing this yet.

[`k3s`]: https://k3s.io
[`pikaur`]: https://github.com/actionless/pikaur
