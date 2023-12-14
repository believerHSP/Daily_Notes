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



