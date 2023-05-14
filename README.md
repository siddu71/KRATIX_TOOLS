# KRATIX_TOOLS

```
#!/bin/bash

# Function to check if the command was successful
check_success() {
  if [ $? -eq 0 ]; then
    printf "\n + $1 installation succeeded\n"
  else
    echo "$1 installation failed"
    exit 1
  fi
}

# INSTALL DOCKER

echo -n "Updating system packages..."
sudo apt-get update -y > /dev/null 2>&1
check_success "System package update"

if dpkg --get-selections | grep -q "^docker.*install$" >/dev/null; then
    echo -n "Removing old versions of Docker..."
    sudo apt-get remove docker docker-engine docker.io containerd runc > /dev/null 2>&1
    check_success "Docker removal"
else
    echo "Docker not installed, skipping removal"
fi

echo -n "Updating system packages again..."
sudo apt-get update -y > /dev/null 2>&1
check_success "System package update"

echo -n "Installing ca-certificates, curl, and gnupg..."
sudo apt-get install ca-certificates curl gnupg -y > /dev/null 2>&1
check_success "Installation of ca-certificates, curl, and gnupg"

echo -n "Setting up Docker repository..."
sudo install -m 0755 -d /etc/apt/keyrings > /dev/null 2>&1
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg > /dev/null 2>&1
sudo chmod a+r /etc/apt/keyrings/docker.gpg > /dev/null 2>&1
check_success "Docker repository setup"

echo -n "Adding Docker repository to apt sources..."
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null 2>&1 

check_success "Adding Docker repository to apt sources"

echo -n "Updating system packages again..."
sudo apt-get update -y > /dev/null 2>&1
check_success "System package update"

echo -n "Installing Docker..."
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y > /dev/null 2>&1
check_success "Docker installation"

echo -n "Running Docker Hello World..."
sudo docker run hello-world > /dev/null 2>&1
check_success "Docker Hello World"

echo -n "Setting Docker permissions..."
sudo chmod 777 /var/run/docker.sock > /dev/null 2>&1
check_success "Setting Docker permissions"

# INSTALL KIND
echo -n "Installing Kind..."
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64 > /dev/null 2>&1 
chmod +x ./kind > /dev/null 2>&1 
sudo mv ./kind /usr/local/bin/kind > /dev/null 2>&1 
check_success "Kind"

# INSTALL KUBECTL
echo -n "Installing Kubectl..."
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" > /dev/null 2>&1 
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256" > /dev/null 2>&1 
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check > /dev/null 2>&1 
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl > /dev/null 2>&1 
check_success "Kubectl"

echo "All installations successful!"
