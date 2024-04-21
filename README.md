# talosctl-compose

Sample config management repository for Talos. Using `rake` to generate `machineconfig` yamls.

## Prerequisites

Ruby's `rake` is mandatory (just `apt install ruby` to have it). `talosctl` can be downloaded using `rake talosctl` task. `helm` is mandatory for helm generator.
`git-crypt` is highly recommended to store your `secrets.yaml`.

# Usage

Create a subfolder in `clusters` folder. Then place there additional configurations bits you like:

* `patches` for `--config-patch`
* `manifests` for `extraManifests`
* `generate` for generated manifests (see example)

Also create `plan.yaml` describing your future cluster:

```yaml
name: dev
assetsBaseUrl: http://matchbox.lan:8080/assets/
version: 1.7.0
kubeVersion: 1.29.3
controlPlane:
  endpoint: https://dev.lan:6443

generate:
- cilium

patches:
- kubeprism

nodes:
  - type: controlplane
    installDisk: /dev/sda
    patches:
    - allow_scheduling_on_control_planes
    - cilium
    manifests:
    - openebs
    machines:
    - ip: 10.1.0.10
      mac: BD:B3:2B:97:3A:E9
  - type: worker
    installDisk: /dev/sda
    machines:
    - ip: 10.1.0.20
      mac: 68:82:6D:19:6D:1C
```

Just run `rake` inside the folder to have `secrets.yaml` and all manifests prepared for deployment to your `matchbox` instance. You can also generate `talosconfig` for the cluster with `rake talosconfig`. 

You can also generate individual bits with `rake assets`, `rake profiles` and `rake groups`. `rake clean` removes the `out` folder.

If you want to have a patch, manifest or whatever shared between clusters, just places them into root folder of this repo. Local version always have precedence over the global one. Same goes with `plan.yaml` declarations.

## Encryption for secrets

Use this setting in your `.gitattributes`:

```
secrets.yaml filter=git-crypt diff=git-crypt
```

Don't forget to install `git-crypt` and to initialize it for this repo.


Have fun!
