# KRATIX_TOOLS

```
#!/bin/bash

# Function to display a spinner while another command is running
spinner() {
  local pid=$1
  local delay=0.1
  local spinstr='|/-\'
  while [ "$(ps a | awk '{print $1}' | grep $pid)" ]; do
    local temp=${spinstr#?}
    printf " [%c]  " "$spinstr"
    local spinstr=$temp${spinstr%"$temp"}
    sleep $delay
    printf "\b\b\b\b\b\b"
  done
  printf "    \b\b\b\b"
}

# Function to check if the command was successful
check_success() {
  if [ $? -eq 0 ]; then
    printf " \e[32m\u2713\e[0m $1 installation succeeded\n"
  else
    echo "$1 installation failed"
    exit 1
  fi
}

# INSTALL DOCKER

echo "Updating system packages..."
(sudo apt-get update -y > /dev/null 2>&1) &
spinner $!
check_success "System package update"

if dpkg --get-selections | grep -q "^docker.*install$" >/dev/null; then
    echo "Removing old versions of Docker..."
    (sudo apt-get remove docker docker-engine docker.io containerd runc > /dev/null 2>&1) &
    spinner $!
    check_success "Docker removal"
else
    echo "Docker not installed, skipping removal"
fi


echo "Updating system packages again..."
(sudo apt-get update -y > /dev/null 2>&1) &
spinner $!
check_success "System package update"

echo "Installing ca-certificates, curl, and gnupg..."
(sudo apt-get install \
    ca-certificates \
    curl \
    gnupg -y > /dev/null 2>&1) &
    spinner $!
check_success "Installation of ca-certificates, curl, and gnupg"

echo "Setting up Docker repository..."
(sudo install -m 0755 -d /etc/apt/keyrings > /dev/null 2>&1
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg > /dev/null 2>&1
sudo chmod a+r /etc/apt/keyrings/docker.gpg > /dev/null 2>&1) &
spinner $!
check_success "Docker repository setup"

echo "Adding Docker repository to apt sources..."
(echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null 2>&1)(sudo apt-get update -y > /dev/null 2>&1) &
spinner $!
check_success "Adding Docker repository to apt sources"

echo "Updating system packages again..."
(sudo apt-get update -y > /dev/null 2>&1) &
spinner $!
check_success "System package update"

echo "Installing Docker..."
(sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y > /dev/null 2>&1) &
spinner $!
check_success "Docker installation"

echo "Running Docker Hello World..."
(sudo docker run hello-world > /dev/null 2>&1) &
spinner $!
check_success "Docker Hello World"

echo "Setting Docker permissions..."
(sudo chmod 777 /var/run/docker.sock > /dev/null 2>&1) &
spinner $!
check_success "Setting Docker permissions"


echo "Installing Docker..."
(sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y > /dev/null 2>&1) &
spinner $!
check_success "Docker"

echo "Running Docker Hello World..."
(sudo docker run hello-world > /dev/null 2>&1) &
spinner $!
check_success "Docker Hello World"

# INSTALL KIND

(echo "Installing Kind..."
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64 > /dev/null 2>&1
chmod +x ./kind > /dev/null 2>&1
sudo mv ./kind /usr/local/bin/kind > /dev/null 2>&1) &
spinner $!
check_success "Kind"

# INSTALL KUBECTL

(echo "Installing Kubectl..."
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" > /dev/null 2>&1
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256" > /dev/null 2>&1
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check > /dev/null 2>&1
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl > /dev/null 2>&1) &
spinner $!
check_success "Kubectl"

echo "All installations successful!"
