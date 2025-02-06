# Incus + Cloud-Init Utilities

This repository contains the following:

1. Incus Data Source for Cloud-Init which is modified from the existing LXD data source

2. Example configuration for Cloud-Init to only use the Incus data source

## Behaviour and Enhancements

The modifications provided in this repository allows stacking of vendor data and user data sections, essentially allow multiple profiles to configure the same module within cloud-init. Currently as of Incus 6.9, configuration is mutually exclusive, meaning for example `runcmd` stanzas defined in vendor data would be overridden by stanzas defined in user data, which is righteous, but stanzas defined within one Profile would be overridden by another Profile.

The crux of the problem is that the default NoCloud data source is primitive compared to the LXD data source, but while the latter allows retrieval and interpolation of all custom keys via Jinja it is still highly inconvenient.

With the changes in this repository applied, one may write the following configuration:

```
architecture: x86_64
config:
  cloud-init.user-data: |
    #cloud-config
    runcmd:
      - echo 3 > /output-from-user-data
  user.custom_2_parent: parent3
  user.user-data.custom1: |
    #cloud-config
    runcmd:
      - echo 1 > /output-custom1
  user.user-data.custom2: |
    ## template: jinja
    #cloud-config
    runcmd:
      - echo {{ ds.config.user_custom_2_parent }} > /output-custom2
```

...and have each runcmd section merged with each other using the default merge strategy of `dict(recurse_array,recurse_str)+list(append)+str(append)` as discussed [here](https://cloudinit.readthedocs.io/en/latest/reference/merging.html). This is achieved by modifying the Incus data source to gather all fragments and generate a MIME Multipart message that replaces the final configuration for both vendor data and user data.

This would also work across multiple profiles given the following example, with the first profile containing:

```
$ incus profile show test-1
config:
  user.user-data.profile1: |
    #cloud-config
    runcmd:
      - echo 1 > /output-from-profile1
```

and the second profile containing:

```
$ incus profile show test-2
config:
  user.user-data.profile2: |
    #cloud-config
    runcmd:
      - echo 1 > /output-from-profile2
```

As long as each profile accrues configuration everything is merged.

Precendence rules are not considered at the moment but this could be added at a later time.

The changes are currently contained in the _MetaDataReader but this can be moved later on.

## Installation

Since the files modify the behaviour of Cloud-Init, it is recommended to place the files into a newly created Incus instance, before it is launched for the first time.

```
$ incus create images:debian/trixie/cloud my-instance
$ incus file push lib/python3/dist-packages/cloudinit/sources/DataSourceIncus.py \
    my-instance/lib/python3/dist-packages/cloudinit/sources/DataSourceIncus.py
$ incus file push etc/cloud/cloud.cfg.d/00_incus.cfg \
    my-instance/etc/cloud/cloud.cfg.d/00_incus.cfg
```

If you would like to update an existing instance then you may clean out all residues of cloud-init:

```
$ incus exec my-instance -- cloud-init clean --logs --machine-id --configs all
$ incus stop my-instance
$ incus start my-instance
```

## Licensing

This repository is made available under the Apache 2.0 License, with portions of this work derived from the [LXD Data Source](https://github.com/canonical/cloud-init/blob/main/cloudinit/sources/DataSourceLXD.py) within [Cloud-Init](https://github.com/canonical/cloud-init) which I elect to license under Apache 2.0.
