# Домашнее задание к занятию «Отказоустойчивость в облаке»

Выполнил: **Нестеренко Андрей**

---

## Цель работы

В ходе выполнения домашнего задания была настроена отказоустойчивая инфраструктура в Yandex Cloud с использованием Terraform.

В результате были выполнены следующие задачи:

* создана облачная сеть и подсеть;
* созданы две идентичные виртуальные машины с помощью аргумента `count`;
* на виртуальные машины установлен и запущен веб-сервер Nginx;
* создана целевая группа для балансировщика;
* в целевую группу добавлены обе виртуальные машины;
* создан сетевой балансировщик нагрузки;
* настроен listener на порту `80`;
* настроен HTTP healthcheck на порту `80`;
* проверено состояние балансировщика и целевой группы в консоли Yandex Cloud;
* выполнен запрос к внешнему IP-адресу балансировщика.

---

## Задание 1

Необходимо было доработать Terraform playbook из предыдущего задания и вместо одной виртуальной машины создать отказоустойчивую конфигурацию из двух виртуальных машин, целевой группы и сетевого балансировщика нагрузки.

### Выполненные действия

С помощью Terraform были созданы:

| Ресурс                            | Описание                                        |
| --------------------------------- | ----------------------------------------------- |
| `yandex_vpc_network`              | Облачная сеть                                   |
| `yandex_vpc_subnet`               | Подсеть в зоне `ru-central1-a`                  |
| `yandex_compute_instance`         | Две виртуальные машины, созданные через `count` |
| `yandex_lb_target_group`          | Целевая группа для балансировщика               |
| `yandex_lb_network_load_balancer` | Сетевой балансировщик нагрузки                  |
| `cloud-init`                      | Установка и запуск Nginx на виртуальных машинах |

---

## Terraform Playbook

### `main.tf`

```hcl
terraform {
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "~> 0.140"
    }
  }

  required_version = ">= 1.0"
}

provider "yandex" {
  token     = var.yc_iam_token
  cloud_id  = var.yc_cloud_id
  folder_id = var.yc_folder_id
  zone      = var.yc_zone
}

data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2204-lts"
}

resource "yandex_vpc_network" "network" {
  name = "netology-ha-network"
}

resource "yandex_vpc_subnet" "subnet" {
  name           = "netology-ha-subnet"
  zone           = var.yc_zone
  network_id     = yandex_vpc_network.network.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_vpc_security_group" "http_sg" {
  name       = "netology-http-sg"
  network_id = yandex_vpc_network.network.id

  ingress {
    description    = "allow-http-from-internet"
    protocol       = "TCP"
    port           = 80
    v4_cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description    = "allow-ssh-from-internet"
    protocol       = "TCP"
    port           = 22
    v4_cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description    = "allow-all-outgoing"
    protocol       = "ANY"
    v4_cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "yandex_compute_instance" "vm" {
  count = 2

  name        = "netology-nginx-${count.index + 1}"
  platform_id = "standard-v1"
  zone        = var.yc_zone

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.id
      size     = 10
      type     = "network-hdd"
    }
  }

  network_interface {
    subnet_id          = yandex_vpc_subnet.subnet.id
    nat                = true
    security_group_ids = [yandex_vpc_security_group.http_sg.id]
  }

  metadata = {
    user-data = templatefile("${path.module}/cloud-init.yaml", {
      ssh_public_key = var.ssh_public_key
      vm_number      = count.index + 1
    })
  }
}

resource "yandex_lb_target_group" "target_group" {
  name = "netology-nginx-target-group"

  dynamic "target" {
    for_each = yandex_compute_instance.vm

    content {
      subnet_id = yandex_vpc_subnet.subnet.id
      address   = target.value.network_interface[0].ip_address
    }
  }
}

resource "yandex_lb_network_load_balancer" "load_balancer" {
  name = "netology-nginx-load-balancer"

  listener {
    name        = "nginx-listener"
    port        = 80
    target_port = 80
    protocol    = "tcp"

    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.target_group.id

    healthcheck {
      name = "nginx-healthcheck"

      http_options {
        port = 80
        path = "/"
      }
    }
  }
}
```

---

### `variables.tf`

```hcl
variable "yc_iam_token" {
  type      = string
  sensitive = true
}

variable "yc_cloud_id" {
  type = string
}

variable "yc_folder_id" {
  type = string
}

variable "yc_zone" {
  type    = string
  default = "ru-central1-a"
}

variable "ssh_public_key" {
  type = string
}
```

---

### `outputs.tf`

```hcl
output "vm_public_ips" {
  value = yandex_compute_instance.vm[*].network_interface[0].nat_ip_address
}

output "vm_private_ips" {
  value = yandex_compute_instance.vm[*].network_interface[0].ip_address
}

output "load_balancer_ip" {
  value = one(flatten([
    for listener in yandex_lb_network_load_balancer.load_balancer.listener : [
      for spec in listener.external_address_spec :
      spec.address
    ]
  ]))
}
```

---

### `cloud-init.yaml`

```yaml
#cloud-config
users:
  - name: yc-user
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ${ssh_public_key}

package_update: true

packages:
  - nginx

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - echo "Hello from nginx VM ${vm_number}" > /var/www/html/index.html
```

---

## Результат выполнения Terraform

После выполнения команды:

```bash
terraform apply
```

были созданы две виртуальные машины, целевая группа и сетевой балансировщик нагрузки.

Вывод Terraform:

```text
Apply complete!

vm_private_ips = [
  "192.168.10.13",
  "192.168.10.34",
]

vm_public_ips = [
  "89.169.158.147",
  "93.77.185.174",
]
```

---

## Проверка балансировщика нагрузки

В веб-консоли Yandex Cloud был проверен созданный сетевой балансировщик нагрузки.

Балансировщик находится в статусе **Active**.

Скриншот статуса балансировщика:

<img src="img/Pasted image 20260609152308.png" width="75%">

---

## Проверка целевой группы

В целевой группе находятся две виртуальные машины:

| Виртуальная машина | Внутренний IP-адрес | Статус    |
| ------------------ | ------------------- | --------- |
| `netology-nginx-1` | `192.168.10.13`     | `Healthy` |
| `netology-nginx-2` | `192.168.10.34`     | `Healthy` |

Скриншот целевой группы:

<img src="img/Pasted image 20260609145821.png" width="50%">

---

## Проверка работы Nginx через балансировщик

Был выполнен запрос на внешний IP-адрес балансировщика:

```text
http://158.160.221.82
```

В результате была получена страница Nginx с ответом от одной из виртуальных машин:

```text
Hello from nginx VM 2
```

Скриншот страницы:

<img src="img/Pasted image 20260609152136.png" width="50%">

---

## Вывод

В ходе выполнения задания была создана отказоустойчивая инфраструктура в Yandex Cloud.

С помощью Terraform были созданы две виртуальные машины с установленным Nginx, целевая группа и сетевой балансировщик нагрузки. Балансировщик принимает входящий трафик на порту `80` и распределяет его между виртуальными машинами. Проверка состояния показала, что обе виртуальные машины находятся в статусе `Healthy`, а балансировщик находится в статусе `Active`.

