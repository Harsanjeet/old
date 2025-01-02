# OKD Deployment Guide for Debian 12

## Prerequisites
1. A Debian 12 server with:
   - Minimum 8 CPU cores
   - 16GB RAM minimum (32GB recommended)
   - 100GB available storage
   - Root or sudo access
   - Static IP address configured
   - Fully qualified domain name (FQDN) configured

2. Required tools:
   - Git
   - Docker or Podman
   - NetworkManager
   - Python 3
   - Access to DNS management for your domain

## Step 1: System Preparation

1. Update the system:
```bash
sudo apt update
sudo apt upgrade -y
```

2. Install required packages:
```bash
sudo apt install -y git python3 python3-pip NetworkManager curl wget jq
```

3. Install container runtime (Docker):
```bash
# Add Docker repository
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian bookworm stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker
```

## Step 2: DNS Configuration

1. Configure your DNS records (replace example.com with your domain):
```
api.okd.example.com       -> Your server IP
*.apps.okd.example.com    -> Your server IP
```

2. Verify DNS resolution:
```bash
dig +short api.okd.example.com
dig +short test.apps.okd.example.com
```

## Step 3: Install OpenShift Client Tools

1. Download and install the `oc` client:
```bash
OC_VERSION=$(curl -s https://api.github.com/repos/openshift/okd/releases/latest | jq -r .tag_name)
wget https://github.com/openshift/okd/releases/download/${OC_VERSION}/openshift-client-linux-${OC_VERSION}.tar.gz
sudo tar xvf openshift-client-linux-${OC_VERSION}.tar.gz -C /usr/local/bin/
sudo chmod +x /usr/local/bin/oc /usr/local/bin/kubectl
```

## Step 4: Configure OKD Installation

1. Create installation directory:
```bash
mkdir ~/okd-install
cd ~/okd-install
```

2. Generate SSH key if not already present:
```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
```

3. Create install configuration:
```bash
cat << EOF > install-config.yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: okd
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 2
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
networking:
  networkType: OpenShiftSDN
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"fake":{"auth": "bar"}}}' 
sshKey: '$(cat ~/.ssh/id_ed25519.pub)'
EOF
```

4. Create installation manifests:
```bash
oc create install-config --dir=~/okd-install
```

## Step 5: Start the Installation

1. Begin the installation:
```bash
openshift-install create cluster --dir=~/okd-install --log-level=info
```

2. Monitor the installation progress (this can take 30-60 minutes):
```bash
tail -f ~/okd-install/.openshift_install.log
```

## Step 6: Post-Installation Configuration

1. Set up authentication:
```bash
export KUBECONFIG=~/okd-install/auth/kubeconfig
```

2. Verify cluster status:
```bash
oc get nodes
oc get co
```

3. Access the web console:
   - URL: https://console-openshift-console.apps.okd.example.com
   - Credentials: Found in ~/okd-install/auth/kubeadmin-password

## Common Issues and Troubleshooting

1. If pods fail to start, check:
```bash
oc get events -A
oc describe pod <pod-name> -n <namespace>
```

2. For networking issues:
```bash
oc get network-attachment-definitions -A
oc get pods -n openshift-sdn
```

3. Check cluster operator status:
```bash
oc get clusteroperators
```

## Maintenance and Updates

1. Check for available updates:
```bash
oc adm upgrade
```

2. Apply updates:
```bash
oc adm upgrade --to-latest=true
```

## Security Considerations

1. Change default passwords
2. Configure identity provider
3. Set up network policies
4. Enable monitoring and logging
5. Regular security patches and updates

Remember to regularly backup etcd and other critical cluster data.
