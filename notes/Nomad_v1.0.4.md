<p align="center">
    <img alt="Nomad" src="https://www.datocms-assets.com/2885/1506458566-blog-nomad.svg" width="35%" />
</p>


#### Basic Setup:

- Download nomad binary & make it executable:

```bash
$ wget https://releases.hashicorp.com/nomad/1.0.4/nomad_1.0.4_linux_amd64.zip
$ unzip nomad_1.0.4_linux_amd64.zip
$ mv nomad /usr/local/bin/
```

- Create configs directory & data directory:

```bash
$ sudo mkdir --parents /opt/nomad
$ sudo mkdir --parents /opt/data/redis
$ sudo mkdir --parents /etc/nomad.d
$ sudo chmod 700 /etc/nomad.d
$ sudo chmod 700 /opt/data

$ sudo touch /etc/nomad.d/nomad.hcl
$ sudo touch /etc/nomad.d/server.hcl
$ sudo touch /etc/nomad.d/client.hcl
```

- Add this configuration to the `/etc/nomad.d/nomad.hcl` configuration file

```bash
datacenter = "dc1"
data_dir = "/opt/nomad"
```

- Add this configuration to the `/etc/nomad.d/client.hcl` configuration file

```hcl
client {
  enabled = true

  host_volume "redis_data" {
    path      = "/opt/data/redis"
    read_only = false
  }
}

plugin "docker" {
  config {
    volumes {
      enabled      = true
      selinuxlabel = "z"
    }
  }
}
```

- Add this configuration to the `/etc/nomad.d/server.hcl` configuration file

```hcl
acl {
  enabled = true
}
server {
  enabled = true
  bootstrap_expect = 1
}
```

- Create a nomad service file

```bash
$ sudo touch /etc/systemd/system/nomad.service
```

- Add this configuration to the nomad service file `/etc/systemd/system/nomad.service`

```bash
[Unit]
Description=Nomad
Documentation=https://nomadproject.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/nomad agent -config /etc/nomad.d
KillMode=process
KillSignal=SIGINT
LimitNOFILE=infinity
LimitNPROC=infinity
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

- Start nomad

```bash
sudo systemctl enable nomad
sudo systemctl start nomad
sudo systemctl status nomad
```

- Genertal ACL secrets

```bash
$ nomad acl bootstrap
Accessor ID  = a8b724ef-b5eb-21c0-1a68-b0d3616e1e62
Secret ID    = 07fe0ee6-59bc-8ea0-aeeb-9b4e9edcb555
Name         = Bootstrap Token
Type         = management
Global       = true
Policies     = n/a
Create Time  = 2021-05-10 21:05:21.498072284 +0000 UTC
Create Index = 10
Modify Index = 10

$ export NOMAD_TOKEN=07fe0ee6-59bc-8ea0-aeeb-9b4e9edcb555
```

- Create your first job to deploy a stateless container from the server ui `http://127.0.0.1:4646/ui`

```hcl
job "api" {
  datacenters = ["dc1"]

  group "example" {
    task "server" {
      driver = "docker"

      config {
        image = "hashicorp/http-echo:0.2.1"

        args = [
          "-listen",
          ":5678",
          "-text",
          "hello world",
        ]
      }

      resources {
        network {
          mbits = 10

          port "http" {
            static = "5678"
          }
        }
      }
    }
  }
}
```

- And another one for stateful container

```hcl
job "cache" {
  datacenters = ["dc1"]

  group "redis" {

    volume "redis_data" {
      type      = "host"
      read_only = false
      source    = "redis_data"
    }

    task "server" {
      driver = "docker"

      volume_mount {
        volume      = "redis_data"
        destination = "/data"
        read_only   = false
      }

      config {
        image = "redis:4-alpine"

        labels = {
          "sh.hippo.service" = "redis"
          "sh.hippo.service_type" = "cache"
        }

        port_map {
          http = 6379
        }

        command = "redis-server"

        args = [
          "--requirepass",
          "mystery",
        ]
      }

      env = {
        HEALTHY_FOR    = -1,
      }

      resources {
        cpu    = 100
        memory = 256

        network {
          mbits = 10

          port "http" {
            static = "6379"
          }
        }
      }
    }
  }
}
```


#### CNI Plugins in Nomad

Nomad uses CNI plugins when bridge networking is used. To install CNI plugins:

```bash
$ curl -L -o cni-plugins.tgz https://github.com/containernetworking/plugins/releases/download/v0.8.0/cni-plugins-linux-amd64-v0.8.0.tgz
$ mkdir -p /opt/cni/bin
$ tar -C /opt/cni/bin -xzf cni-plugins.tgz
```

Task with bridge networking

```hcl
job "clivern" {
  datacenters = ["dc1"]

  group "services" {

    network {
      mode = "bridge"

      port "toad0_port" {
        to = 8080
      }

      port "toad1_port" {
        to = 8081
      }
    }

    task "toad0" {
      driver = "docker"

      config {
        image = "clivern/toad:release-0.2.4"

        labels = {
          "com.clivern.service" = "toad"
          "com.clivern.service_type" = "web"
        }

        ports = ["toad0_port"]

        command = "./toad"

        args = [
          "--port",
          "${NOMAD_PORT_toad0_port}",
        ]
      }

      env = {
        IS_STATEFUL    = "false",
      }

      resources {
        network {
          mbits = 10
        }
      }
    }

    task "toad1" {
      driver = "docker"

      config {
        image = "clivern/toad:release-0.2.3"

        labels = {
          "com.clivern.service" = "toad"
          "com.clivern.service_type" = "web"
        }

        ports = ["toad1_port"]

        command = "./toad"

        args = [
          "--port",
          "${NOMAD_PORT_toad1_port}",
        ]
      }

      env = {
        IS_STATEFUL    = "false",
      }

      resources {
        network {
          mbits = 10
        }
      }
    }
  }
}
```

```hcl
job "clivern" {
  datacenters = ["dc1"]

  group "services" {

    network {

      port "toad0_port" {
        static = 8080
      }

      port "toad1_port" {
        static = 8081
      }
    }

    task "toad0" {
      driver = "docker"

      config {
        image = "clivern/toad:release-0.2.4"

        labels = {
          "com.clivern.service" = "toad"
          "com.clivern.service_type" = "web"
        }

        ports = ["toad0_port"]

        command = "./toad"

        args = [
          "--port",
          "${NOMAD_PORT_toad0_port}",
        ]
      }

      env = {
        IS_STATEFUL    = "false",
      }

      resources {
        network {
          mbits = 10
        }
      }
    }

    task "toad1" {
      driver = "docker"

      config {
        image = "clivern/toad:release-0.2.3"

        labels = {
          "com.clivern.service" = "toad"
          "com.clivern.service_type" = "web"
        }

        ports = ["toad1_port"]

        command = "./toad"

        args = [
          "--port",
          "${NOMAD_PORT_toad1_port}",
        ]
      }

      env = {
        IS_STATEFUL    = "false",
      }

      resources {
        network {
          mbits = 10
        }
      }
    }
  }
}
```


#### References:

- [Nomad Docs](https://www.nomadproject.io/docs)
- [Learn Nomad](https://learn.hashicorp.com/collections/nomad/get-started)
