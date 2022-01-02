---
title: "terraform moved block を使ってみる"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform"]
published: true
---

terraform v1.1.0 から、moved ブロックが使えるようになりました。

https://www.terraform.io/language/modules/develop/refactoring

これにより、terraform リソース名の変更を明示的にコードで管理できるようになります。

terraform v1.1.0, v1.1.1 はバグが報告されているので、バージョン v1.1.2 を使用します。

https://github.com/hashicorp/terraform/releases/tag/v1.1.2

## moved block は何が嬉しいのか

terraform のコードが煩雑になってくると、以下のような要望が出てきます。

- 初期に作成したリソースをリファクタリングしたい
  - `module` を使って一貫性のあるリソースを作りたい
  - `count` や `for_each` を使って DRY なコードにしたい

terraform で管理しているリソース名を変更する場合、tf ファイルだけでなく tfstate ファイルも変更する必要があります。

これまでは `terraform state mv` コマンドを実行して、tfstate に保存されたリソース名を **1 つ 1 つ**変更していました。
これからは moved block でリソース名変更をコードで宣言することにより、コードレビューの実施や `terraform apply` コマンドから一括でリソース名を変更できます。

## サンプルコード

terraform で管理している自宅の ESXi をリファクタリングしてみます。

ソースコードは以下になります。

https://github.com/ymmmtym/home

### リファクタリングする tf ファイル

とてもシンプルですが、以下の tf ファイルをリファクタリングしてみます。

```hcl:guest-vyos.tf
resource "esxi_guest" "vyos01" {
  guest_name     = "vyos01"
  power          = "on"
  disk_store     = "diskstore01"
  clone_from_vm  = "template-vyos-rolling"
  memsize        = "1024"
  numvcpus       = "1"
  boot_disk_size = "10"
  network_interfaces {
    virtual_network = var.VIRTUAL_NETWORK
    mac_address     = "00:50:56:00:00:21"
  }
  network_interfaces {
    virtual_network = esxi_portgroup.portgroup100_1.name
    mac_address     = "00:50:56:00:64:01"
  }
  network_interfaces {
    virtual_network = esxi_portgroup.portgroup101_1.name
    mac_address     = "00:50:56:00:65:01"
  }
}

resource "esxi_guest" "vyos02" {
  guest_name     = "vyos02"
  power          = "on"
  disk_store     = "diskstore01"
  clone_from_vm  = "template-vyos-rolling"
  memsize        = "1024"
  numvcpus       = "1"
  boot_disk_size = "10"
  network_interfaces {
    virtual_network = var.VIRTUAL_NETWORK
    mac_address     = "00:50:56:00:00:22"
  }
  network_interfaces {
    virtual_network = esxi_portgroup.portgroup100_1.name
    mac_address     = "00:50:56:00:64:02"
  }
  network_interfaces {
    virtual_network = esxi_portgroup.portgroup101_1.name
    mac_address     = "00:50:56:00:65:02"
  }
}
```

ESXi に 2 つの VM(vyos) を定義した tf ファイルです。

### tf ファイルのリファクタリング

`locals` や `count` を使ってリファクタリングしてみます。

```hcl:guest-vyos.tf
locals {
  vyos = {
    numvcups       = 1
    memsize        = 1024
    boot_disk_size = 10
    nodes          = 2
  }
}

resource "esxi_guest" "vyos" {
  count          = local.vyos.nodes
  guest_name     = "vyos${format("%02d", count.index + 1)}"
  power          = "on"
  disk_store     = var.DISK_STORE
  clone_from_vm  = "template-vyos-rolling"
  memsize        = local.vyos.memsize
  numvcpus       = local.vyos.numvcups
  boot_disk_size = local.vyos.boot_disk_size
  network_interfaces {
    virtual_network = var.VIRTUAL_NETWORK
    mac_address     = "00:50:56:00:00:${format("%x", count.index + 33)}"
  }
  network_interfaces {
    virtual_network = esxi_portgroup.portgroup100_1.name
    mac_address     = "00:50:56:00:64:${format("%02d", count.index + 1)}"
  }
  network_interfaces {
    virtual_network = esxi_portgroup.portgroup101_1.name
    mac_address     = "00:50:56:00:65:${format("%02d", count.index + 1)}"
  }
}
```

`count` を使って `resource` ブロックを 1 つにまとめました。
また必要な変数は `locals` や `count` から取得して既存の状態から変更されないようにしています。

このとき、リソース名は以下のように変更されます。

| 変更前 | 変更後 |
| --- | --- |
| `esxi_guest.vyos01` | `esxi_guest.vyos.0` |
| `esxi_guest.vyos02` | `esxi_guest.vyos.1` |

既存のリソースが削除され、新規リソースが作成されたので、 `terraform plan` を実行すると created/destroyed が発生します。

```bash
$ terraform plan
# snip...

  # esxi_guest.vyos[0] will be created
  + resource "esxi_guest" "vyos" {
      + boot_disk_size         = "10"
      + boot_disk_type         = "thin"
      + clone_from_vm          = "template-vyos-rolling"
      + disk_store             = "datastore1"
      + guest_name             = "vyos01"
      + guest_shutdown_timeout = (known after apply)
      + guest_startup_timeout  = (known after apply)
      + guestos                = (known after apply)
      + id                     = (known after apply)
      + ip_address             = (known after apply)
      + memsize                = "1024"
      + notes                  = (known after apply)
      + numvcpus               = "1"
      + ovf_properties_timer   = (known after apply)
      + power                  = "on"
      + resource_pool_name     = "/"
      + virthwver              = (known after apply)

      + network_interfaces {
          + mac_address     = "00:50:56:00:00:21"
          + nic_type        = (known after apply)
          + virtual_network = "VM Network"
        }
      + network_interfaces {
          + mac_address     = "00:50:56:00:64:01"
          + nic_type        = (known after apply)
          + virtual_network = "vSwitch1 PortGroup100"
        }
      + network_interfaces {
          + mac_address     = "00:50:56:00:65:01"
          + nic_type        = (known after apply)
          + virtual_network = "vSwitch1 PortGroup101"
        }
    }

  # esxi_guest.vyos[1] will be created
  + resource "esxi_guest" "vyos" {
      + boot_disk_size         = "10"
      + boot_disk_type         = "thin"
      + clone_from_vm          = "template-vyos-rolling"
      + disk_store             = "datastore1"
      + guest_name             = "vyos02"
      + guest_shutdown_timeout = (known after apply)
      + guest_startup_timeout  = (known after apply)
      + guestos                = (known after apply)
      + id                     = (known after apply)
      + ip_address             = (known after apply)
      + memsize                = "1024"
      + notes                  = (known after apply)
      + numvcpus               = "1"
      + ovf_properties_timer   = (known after apply)
      + power                  = "on"
      + resource_pool_name     = "/"
      + virthwver              = (known after apply)

      + network_interfaces {
          + mac_address     = "00:50:56:00:00:22"
          + nic_type        = (known after apply)
          + virtual_network = "VM Network"
        }
      + network_interfaces {
          + mac_address     = "00:50:56:00:64:02"
          + nic_type        = (known after apply)
          + virtual_network = "vSwitch1 PortGroup100"
        }
      + network_interfaces {
          + mac_address     = "00:50:56:00:65:02"
          + nic_type        = (known after apply)
          + virtual_network = "vSwitch1 PortGroup101"
        }
    }

  # esxi_guest.vyos01 will be destroyed
  # (because esxi_guest.vyos01 is not in configuration)
  - resource "esxi_guest" "vyos01" {
      - boot_disk_size         = "10" -> null
      - boot_disk_type         = "thin" -> null
      - clone_from_vm          = "template-vyos-rolling" -> null
      - disk_store             = "datastore1" -> null
      - guest_name             = "vyos01" -> null
      - guest_shutdown_timeout = 20 -> null
      - guest_startup_timeout  = 120 -> null
      - guestos                = "debian6-64" -> null
      - id                     = "437" -> null
      - ip_address             = "192.168.0.4" -> null
      - memsize                = "1024" -> null
      - numvcpus               = "1" -> null
      - ovf_properties_timer   = 0 -> null
      - power                  = "on" -> null
      - resource_pool_name     = "/" -> null
      - virthwver              = "9" -> null

      - network_interfaces {
          - mac_address     = "00:50:56:00:00:21" -> null
          - nic_type        = "e1000" -> null
          - virtual_network = "VM Network" -> null
        }
      - network_interfaces {
          - mac_address     = "00:50:56:00:64:01" -> null
          - nic_type        = "e1000" -> null
          - virtual_network = "vSwitch1 PortGroup100" -> null
        }
      - network_interfaces {
          - mac_address     = "00:50:56:00:65:01" -> null
          - nic_type        = "e1000" -> null
          - virtual_network = "vSwitch1 PortGroup101" -> null
        }
    }

  # esxi_guest.vyos02 will be destroyed
  # (because esxi_guest.vyos02 is not in configuration)
  - resource "esxi_guest" "vyos02" {
      - boot_disk_size         = "10" -> null
      - boot_disk_type         = "thin" -> null
      - clone_from_vm          = "template-vyos-rolling" -> null
      - disk_store             = "datastore1" -> null
      - guest_name             = "vyos02" -> null
      - guest_shutdown_timeout = 20 -> null
      - guest_startup_timeout  = 120 -> null
      - guestos                = "debian6-64" -> null
      - id                     = "438" -> null
      - ip_address             = "192.168.0.34" -> null
      - memsize                = "1024" -> null
      - numvcpus               = "1" -> null
      - ovf_properties_timer   = 0 -> null
      - power                  = "on" -> null
      - resource_pool_name     = "/" -> null
      - virthwver              = "9" -> null

      - network_interfaces {
          - mac_address     = "00:50:56:00:00:22" -> null
          - nic_type        = "e1000" -> null
          - virtual_network = "VM Network" -> null
        }
      - network_interfaces {
          - mac_address     = "00:50:56:00:64:02" -> null
          - nic_type        = "e1000" -> null
          - virtual_network = "vSwitch1 PortGroup100" -> null
        }
      - network_interfaces {
          - mac_address     = "00:50:56:00:65:02" -> null
          - nic_type        = "e1000" -> null
          - virtual_network = "vSwitch1 PortGroup101" -> null
        }
    }
```

### moved ブロックの作成

`moved.tf` ファイルを作成します。2 つのリソース名を変更したので 2 ブロック分作成します。[^1]

```bash
moved {
  from = esxi_guest.vyos01
  to   = esxi_guest.vyos.0
}

moved {
  from = esxi_guest.vyos02
  to   = esxi_guest.vyos.1
}
```

もう一度、`terraform plan` を実行してみます。

```bash
$ terraform plan
# snip...

  # esxi_guest.vyos01 has moved to esxi_guest.vyos[0]
    resource "esxi_guest" "vyos" {
        id                     = "437"
        # (15 unchanged attributes hidden)

        # (3 unchanged blocks hidden)
    }

  # esxi_guest.vyos02 has moved to esxi_guest.vyos[1]
    resource "esxi_guest" "vyos" {
        id                     = "438"
        # (15 unchanged attributes hidden)

        # (3 unchanged blocks hidden)
    }
```

`# <旧リソース名> has moved to <新リソース名>` となり、 terraform 側でリソース名変更が認識されています。また、このとき tfstate はまだ変更されていません。

```bash
$ terraform state list
# snip...

esxi_guest.vyos01
esxi_guest.vyos02
```

### terraform apply の実行

この状態で `terraform apply` すると、tfstate の内容も書き変わります。
apply した後に `terraform state list` コマンドを実行すると変更されたことが確認できます。

```bash
$ terraform state list
# snip...

esxi_guest.vyos[0]
esxi_guest.vyos[1]
```

tfstate の書き換えが完了したら `moved.tf` ファイルは不要なので削除して完了です。

## さいごに

今回のサンプルコードでは 2 つだけなので、moved block を使わずに CLI で実施する場合は以下のコマンドでリソース名を変更できます。

```bash
terraform state mv esxi_guest.vyos01 esxi_guest.vyos.0
terraform state mv esxi_guest.vyos02 esxi_guest.vyos.1
```

変更するリソース量が多くなったり、コードレビューが必要な場面で活用できそうです。

[^1]: `to` に module も指定できます。
