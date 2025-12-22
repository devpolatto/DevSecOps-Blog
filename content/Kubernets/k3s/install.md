---
title: Instalação
tags:
  - Kubernets
  - k3s
enableToc: true
---
```shell
#!/bin/bash

# Variables
DOMAIN_NAME="k3s.lab.local"  # Replace with your preferred domain for the K3s cluster
INSTALL_OPTIONS="--write-kubeconfig-mode 644 --tls-san ${DOMAIN_NAME}"
K3S_BIN_DIR="/usr/local/bin"
K3S_SERVICE_FILE="/etc/systemd/system/k3s.service"

# Step 1: Install K3s
echo "Installing K3s..."
curl -sfL https://get.k3s.io | sh -s - ${INSTALL_OPTIONS}

# Step 2: Configure domain name (optional)
# Adds entry to /etc/hosts for the cluster domain to resolve locally
echo "Configuring domain name..."
if ! grep -q "${DOMAIN_NAME}" /etc/hosts; then
    echo "127.0.0.1 ${DOMAIN_NAME}" | sudo tee -a /etc/hosts > /dev/null
fi

# Step 3: Enable and start K3s as a systemd service
echo "Setting up K3s as a systemd service..."
sudo systemctl enable k3s
sudo systemctl start k3s

# Step 4: Check K3s service status
if sudo systemctl status k3s --no-pager; then
    echo "K3s installed and running successfully!"
    echo "To access the cluster, use the kubeconfig file at /etc/rancher/k3s/k3s.yaml"
    echo "Use the domain name ${DOMAIN_NAME} to access the API server if configured in kubeconfig"
else
    echo "K3s installation failed or the service did not start. Check the logs with 'sudo journalctl -u k3s'"
    exit 1
fi

# Step 5: Print post-install information
echo "Configuration complete. You may now use kubectl to interact with the cluster."
echo "For example:"
echo "  export KUBECONFIG=/etc/rancher/k3s/k3s.yaml"
echo "  kubectl get nodes"

```