#!/usr/bin/env bash

###############################################################################
# Updated Docker installation script using Docker's official instructions
# plus best practices (group management, Kind, and Kubectl).
###############################################################################

set -euo pipefail
trap 'echo "An unexpected error occurred. Exiting..." >&2; exit 1;' ERR

#######################################
# Helper Functions
#######################################
log_info() {
  echo -e "\n[INFO] $1"
}

check_success() {
  if [ $? -eq 0 ]; then
    echo -e "[SUCCESS] $1"
  else
    echo -e "[ERROR] $1"
    exit 1
  fi
}

#######################################
# Ensure the script is run with root privileges
#######################################
if [ "$EUID" -ne 0 ]; then
  echo "[ERROR] Please run this script as root or with sudo."
  exit 1
fi

# Identify the user who invoked sudo, or fallback to $USER
CURRENT_USER="${SUDO_USER:-$USER}"

###############################################################################
# 1) Remove old Docker-related packages (using Docker's snippet)
###############################################################################
log_info "Removing old Docker-related packages..."
for pkg in docker.io docker-doc docker-compose docker-compose-v2 \
           podman-docker containerd runc; do
  apt-get remove -y "$pkg" >/dev/null 2>&1 || true
done
check_success "Removal of old packages"

###############################################################################
# 2) Update system packages, install dependencies for Docker
###############################################################################
log_info "Updating system packages..."
apt-get update -y >/dev/null 2>&1
check_success "System package update"

log_info "Installing dependencies (ca-certificates, curl)..."
apt-get install -y ca-certificates curl >/dev/null 2>&1
check_success "Dependencies installed"

###############################################################################
# 3) Add Docker's official GPG key
###############################################################################
log_info "Adding Docker's official GPG key..."
install -m 0755 -d /etc/apt/keyrings >/dev/null 2>&1
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc >/dev/null 2>&1
chmod a+r /etc/apt/keyrings/docker.asc
check_success "Docker GPG key added"

###############################################################################
# 4) Set up the Docker repository
###############################################################################
log_info "Adding Docker repository to apt sources..."
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list
check_success "Docker repository added"

###############################################################################
# 5) Update package lists and install Docker Engine
###############################################################################
log_info "Updating system packages again..."
apt-get update -y >/dev/null 2>&1
check_success "System package update"

log_info "Installing Docker CE (and related components)..."
apt-get install -y docker-ce docker-ce-cli containerd.io \
                   docker-buildx-plugin docker-compose-plugin >/dev/null 2>&1
check_success "Docker installation"

###############################################################################
# 6) Test Docker with hello-world
###############################################################################
log_info "Testing Docker..."
docker run hello-world >/dev/null 2>&1
check_success "Docker Hello World test"

###############################################################################
# 7) Properly set Docker group and permissions
###############################################################################
log_info "Creating or verifying 'docker' group..."
if ! getent group docker >/dev/null 2>&1; then
  groupadd docker
  check_success "Created docker group"
else
  echo "[INFO] 'docker' group already exists"
fi

# Add the current user to the docker group (unless already present)
if id -nG "$CURRENT_USER" | grep -qw "docker"; then
  echo "[INFO] User '$CURRENT_USER' is already in the 'docker' group"
else
  usermod -aG docker "$CURRENT_USER"
  check_success "Added user '$CURRENT_USER' to 'docker' group"
fi

# Adjust the Docker socket permissions
log_info "Adjusting permissions on /var/run/docker.sock..."
chown root:docker /var/run/docker.sock || true
chmod 660 /var/run/docker.sock || true
check_success "Docker socket permissions set to 'root:docker' with mode 660"

###############################################################################
# 8) (Optional) Install Kind
###############################################################################
log_info "Installing Kind..."
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64 >/dev/null 2>&1
chmod +x ./kind >/dev/null 2>&1
mv ./kind /usr/local/bin/kind >/dev/null 2>&1
check_success "Kind installation"

###############################################################################
# 9) (Optional) Install Kubectl
###############################################################################
log_info "Installing Kubectl..."
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
  >/dev/null 2>&1
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256" \
  >/dev/null 2>&1

echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check >/dev/null 2>&1
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl >/dev/null 2>&1
check_success "Kubectl installation"

###############################################################################
# 10) Final Notices
###############################################################################
echo -e "\n[SUCCESS] All installations completed successfully!"
echo "[INFO] If you are not root, please log out and log back in (or run 'newgrp docker')"
echo "[INFO] to use Docker without sudo under user '$CURRENT_USER'."
