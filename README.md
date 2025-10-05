<p align="center">
  <img src="https://raw.githubusercontent.com/volantvm/volant/main/banner.png" alt="Nginx Plugin for Volant"/>
</p>

<p align="center">
  <a href="https://github.com/volantvm/oci-plugin-example/actions">
    <img src="https://img.shields.io/github/actions/workflow/status/volantvm/oci-plugin-example/release.yml?branch=main&style=flat-square&label=build" alt="Build Status">
  </a>
  <a href="https://github.com/volantvm/oci-plugin-example/releases">
    <img src="https://img.shields.io/github/v/release/volantvm/oci-plugin-example.svg?style=flat-square" alt="Latest Release">
  </a>
  <a href="https://github.com/volantvm/oci-plugin-example/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/License-Apache_2.0-black.svg?style=flat-square" alt="License">
  </a>
</p>

---

# Nginx Plugin for Volant

Official [Nginx](https://nginx.org/) web server plugin for [Volant](https://github.com/volantvm/volant) microVMs.

This repository serves as both a **working plugin** and a **reference implementation** for OCI rootfs plugin authors.

---

## Quick Install

Install this plugin directly from GitHub:

```bash
volar plugins install --manifest https://github.com/volantvm/oci-plugin-example/releases/latest/download/nginx.json
```

Create and run an Nginx microVM:

```bash
volar vms create my-nginx --plugin nginx --cpu 1 --memory 1024
curl http://192.168.127.10
```

---

## What is This?

This is an **OCI rootfs-based plugin** that packages:
- Complete Nginx web server (from official Docker image)
- Full filesystem with all dependencies
- Packaged as a bootable ext4 image (~64MB)

The plugin boots in **2-5 seconds** and provides a full Linux environment.

---

## Repository Structure

```
oci-plugin-example/
├── .github/workflows/
│   └── release.yml    # GitHub Actions for reproducible builds
├── manifest/
│   └── nginx.json     # Plugin manifest (install from this)
├── fledge.toml        # Build configuration
└── README.md          # This file
```

---

## For Plugin Users

### Install the Plugin

```bash
# Install from GitHub
volar plugins install --manifest https://raw.githubusercontent.com/volantvm/oci-plugin-example/main/manifest/nginx.json

# Verify installation
volar plugins list
```

### Create a VM

```bash
# Create a VM with default settings
volar vms create my-nginx --plugin nginx --cpu 1 --memory 1024

# Check VM status
volar vms list

# Test the server
curl http://192.168.127.10
```

### Customize Configuration

If you need custom Nginx configuration:

1. Fork this repository
2. Edit `fledge.toml` (add [mappings] for configs)
3. Rebuild using the workflow (or locally with `fledge build`)
4. Install your custom plugin

---

## For Plugin Authors

This repository demonstrates **best practices** for building OCI rootfs plugins:

### 1. Reproducible Builds

The GitHub Actions workflow:
- Downloads fledge binary from official releases
- Builds bootable ext4 image with `fledge build`
- Calculates checksums automatically
- Creates GitHub releases with artifacts

### 2. Security & Trust

- Uses official Docker Hub images
- SHA256 checksums in the manifest
- Transparent build process anyone can audit

### 3. Proper Structure

```toml
# fledge.toml
[plugin]
name = "nginx"

version = "1"
strategy = "oci_rootfs"

[source]
image = "nginx:alpine"

[filesystem]
type = "ext4"
size_buffer_mb = 100
preallocate = false
```

**Breakdown** (from fledge docs):
- `[plugin] name = "nginx"`: Sets the output filename prefix (nginx-rootfs.img).
- `strategy = "oci_rootfs"`: Builds from Docker/OCI image to ext4 rootfs.
- `[source] image = "nginx:alpine"`: Source Docker image to convert.
- `[filesystem]`: Configures ext4 output (type, buffer size, preallocation).

### 4. Clean Manifest

```json
{
  "schema_version": "1.0",
  "name": "nginx",
  "version": "0.1.0",
  "runtime": "nginx",
  "enabled": true,
  "image": "nginx:alpine",
  "rootfs": {
    "url": "https://github.com/volantvm/oci-plugin-example/releases/download/v0.1.0/nginx-rootfs.img",
    "checksum": "sha256:..."
  },
  "resources": {
    "cpu_cores": 1,
    "memory_mb": 1024
  },
  "workload": {
    "type": "http",
    "entrypoint": ["/docker-entrypoint.sh", "nginx", "-g", "daemon off;"],
    "base_url": "http://127.0.0.1:80",
    "workdir": "/",
    "env": {
      "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    }
  },
  "health_check": {
    "endpoint": "/",
    "timeout_ms": 10000
  }
}
```

---

## Building Locally

### Prerequisites

- Linux with KVM support (for testing)
- Docker installed (for pulling images)

### Build Steps

```bash
# 1. Clone the repository
git clone https://github.com/volantvm/oci-plugin-example
cd oci-plugin-example

# 2. Install fledge (binary download from README)
curl -LO https://github.com/volantvm/fledge/releases/latest/download/fledge-linux-amd64
chmod +x fledge-linux-amd64 && sudo mv fledge-linux-amd64 /usr/local/bin/fledge

# 3. Build the rootfs
sudo fledge build
# Outputs: nginx-rootfs.img + nginx.manifest.json

# 4. Calculate checksum
sha256sum nginx-rootfs.img

# 5. Update manifest with local path (for testing)
# Edit manifest/nginx.json: set "url" to full path of nginx-rootfs.img, "checksum" to SHA256

# 6. Install locally
volar plugins install --manifest manifest/nginx.json

# 7. Test
volar vms create test-nginx --plugin nginx --cpu 1 --memory 1024
curl http://192.168.127.10
```

---

## Creating Your Own Plugin

Use this repository as a template:

### 1. Fork or Copy This Repository

```bash
git clone https://github.com/volantvm/oci-plugin-example my-plugin
cd my-plugin
```

### 2. Modify for Your Application

**Update `fledge.toml`:**
```toml
[plugin]
name = "myapp"

[source]
image = "myapp:alpine"

# Optional mappings for custom files
[mappings]
"./config.yaml" = "/etc/myapp/config.yaml"
```

**Update `manifest/myapp.json`:**
- Change `name`, `version`, `runtime`
- Update `image` to your Docker image
- Update `entrypoint` to your app command
- Adjust `resources`, `base_url`, `health_check`

### 3. Test Locally First

```bash
sudo fledge build
volar plugins install --manifest manifest/myapp.json
volar vms create test --plugin myapp
```

### 4. Push and Tag

```bash
git add .
git commit -m "Initial plugin version"
git tag v0.1.0
git push origin main --tags
```

GitHub Actions will automatically:
- Build the rootfs image
- Calculate checksums
- Create a release
- Publish the manifest

### 5. Users Install Your Plugin

```bash
volar plugins install --manifest https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_PLUGIN/main/manifest/myapp.json
volar vms create my-vm --plugin myapp
```

---

## Troubleshooting

### Plugin Won't Install

```bash
# Check if manifest is valid JSON
cat manifest/nginx.json | jq .

# Verify checksum matches
sha256sum nginx-rootfs.img
```

### VM Won't Start

```bash
# Check VM logs
volar vms logs my-nginx

# Check VM status
volar vms list

# Try with more memory
volar vms create my-nginx --plugin nginx --cpu 1 --memory 2048
```

### Health Check Fails

The health check polls `http://127.0.0.1:80/` inside the VM. Make sure:
- Nginx is listening on port 80
- The entrypoint is correct
- The app binds to 0.0.0.0

### Build Fails

```bash
# Check Docker is running
docker ps

# Verify image pull
docker pull nginx:alpine

# Check fledge
fledge --version

# Run with sudo if needed
sudo fledge build
```

---

## Resources

- [Volant Documentation](https://github.com/volantvm/volant)
- [Fledge Build Tool](https://github.com/volantvm/fledge)
- [Nginx Web Server](https://nginx.org/)
- [Plugin Development Guide](https://github.com/volantvm/volant/blob/main/docs/4_plugin-development/3_oci-rootfs.md)
- [Initramfs Plugin Example](https://github.com/volantvm/initramfs-plugin-example)

---

## License

This plugin is licensed under the **Apache License 2.0** - See [LICENSE](LICENSE) for details.

Nginx is licensed under the 2-clause BSD license by F5, Inc.

---

**Copyright © 2025 Volant VM**
