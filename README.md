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
script in `scripts`.

- Know the interface you want to bind to (e.g. the name of your wifi card from
a command like `ip a`)
- Create a `k3s_hosts` file with newline separate hostnames
- Backup /etc/hosts to your `$HOME`, since there are no guardrails on this script

```shell
sudo ./scripts/update_k3s_hosts -i wlp2s0 k3s_hosts
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

`k3s` by default runs on containerd instead of docker so getting images 
running in the cluster isn't quite so straightforward. The `publish-local`
script is a wrapper around docker and k3s/ctr to import images for use inside
the cluster, simply pass it the image name and it will build the Dockerfile
in the current directory:

```sh
./scripts/publish-local my.domain.com/container:latest
kc run -i --tty my-container --image=my.domain.com/container:latest --image-pull-policy=Never -- sh
```

[`k3s`]: https://k3s.io
[`pikaur`]: https://github.com/actionless/pikaur
