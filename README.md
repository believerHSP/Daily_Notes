# Deriving Resource Requirements: 
# The Aprroach to derive the compute, Network & storage requirement for any setup is, calculating for: 
  OS requirements;
  Cluster requirements; RKE2, K3s on top of OS.
  Application requirements; DB server, Rancher server on top of Cluster.

# Note: Always add upto 20%-25% more resources than the above calculated number. (Anas Sir)
  
# K8s Cluster Resources requirement suggestion by Sandeep Sir:
  Master: 100 GB Disk per Node
  Worker: 200-300 GB Disk per Node

# Disk space  utilization of a Rancher Container:
  df -h 
  du -h
  
# Disk Size of a Host Machine where Rancher Server is running: 
  lsblk
  fdisk -l

# K3s Networking:
 1. outbound connections: 
 2. reverse tunneling:
 3. Kubelet traffic:
 4. Metrics server: 

# Tracing the path of network traffic in Kubernetes:
  https://learnk8s.io/kubernetes-network-packets

# 18 Dec'23

Reference: https://ranchermanager.docs.rancher.com/v2.5/how-to-guides/new-user-guides/backup-restore-and-disaster-recovery/migrate-rancher-to-new-cluster
Understand this link in detail as its the main component utilized in migration: backup-restore-operator: https://github.com/rancher/backup-restore-operator

# 1. Install the rancher-backup Helm chart:
   
Install version 1.x.x of the rancher-backup chart. The following assumes a connected environment with access to DockerHub:
# Figure out the specific version I need to install ??       //

    helm repo add rancher-charts https://charts.rancher.io
    helm repo update
    helm install rancher-backup-crd rancher-charts/rancher-backup-crd -n cattle-resources-system --create-namespace --version $CHART_VERSION
    helm install rancher-backup rancher-charts/rancher-backup -n cattle-resources-system --version $CHART_VERSION

Brief: https://ranchermanager.docs.rancher.com/pages-for-subheaders/backup-restore-and-disaster-recovery#installing-the-rancher-backup-operator

# 2. Restore from backup using a Restore custom resource: 





























