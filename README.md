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

1. This operator provides ability to backup and restore Kubernetes applications (metadata) running on any cluster.
2. It accepts a list of resources that need to be backed up for a particular application. It then gathers these resources by querying the Kubernetes API server,
   packages all the resources to create a tarball file and pushes it to the configured backup storage location.
# 3. Since it gathers resources by querying the API server, it can back up applications from any type of Kubernetes cluster. 
# 4. The operator preserves the ownerReferences on all resources, hence maintaining dependencies between objects.

# Branches and Releases: 
  the tag v3.x.x is cut from the release/v3.0 branch for Rancher v2.7.x line
  the tag v2.x.x is cut from the release/v2.0 branch for Rancher v2.6.x line
  the tag v1.x.x is cut from the release/v1.0 branch for Rancher v2.5.x line



# 1. Install the rancher-backup Helm chart:
   
Install version 1.x.x of the rancher-backup chart. The following assumes a connected environment with access to DockerHub:
# Figure out the specific version I need to install ??       //

    helm repo add rancher-charts https://charts.rancher.io
    helm repo update
    helm install rancher-backup-crd rancher-charts/rancher-backup-crd -n cattle-resources-system --create-namespace --version $CHART_VERSION
    helm install rancher-backup rancher-charts/rancher-backup -n cattle-resources-system --version $CHART_VERSION

Brief: https://ranchermanager.docs.rancher.com/pages-for-subheaders/backup-restore-and-disaster-recovery#installing-the-rancher-backup-operator

# 2. Restore from backup using a Restore custom resource: 

     Create a restore custom resource yaml and apply it.
     
     # migrationResource.yaml
        apiVersion: resources.cattle.io/v1
        kind: Restore
        metadata:
          name: restore-migration
        spec:
          backupFilename: backup-b0450532-cee1-4aa1-a881-f5f48a007b1c-2020-09-15T07-27-09Z.tar.gz
          prune: false
          encryptionConfigSecretName: encryptionconfig
          storageLocation:
            s3:
              credentialSecretName: s3-creds
              credentialSecretNamespace: default
              bucketName: backup-test
              folder: ecm1
              region: us-west-2
              endpoint: s3.us-west-2.amazonaws.com

      # kubectl apply -f migrationResource.yaml

# 3. Install cert-manager: 
     https://ranchermanager.docs.rancher.com/v2.5/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster#4-install-cert-manager
     
     

# 4. Bring up Rancher with Helm
     Use the same version of Helm to install Rancher, that was used on the first cluster.

    helm install rancher rancher-latest/rancher \
      --namespace cattle-system \
      --set hostname=<same hostname as the server URL from the first Rancher server> \





# Backing up the present Rancher using BRO: 
https://ranchermanager.docs.rancher.com/v2.5/pages-for-subheaders/backup-restore-and-disaster-recovery#changes-in-rancher-v25

1.Stop the RAncher-server container:
  docker stop rancher-server-name

2.Create a Data-container.
docker create --volumes-from rancher_v2.6.13 --name rancher-data-data-10-25-2023 rancher/rancher:v2.6.13

3.Create a backup tarball from the above data container:
docker run --name busybox-backup-v2.6.13 --volumes-from rancher-data-data-10-25-2023 -v $PWD:/backup:z busybox tar pzcvf /backup/rancher-data-backup-v2.6.13-10-25-2023.tar.gz /var/lib/rancher

4.ls & check for the backup tarball.

5.Remove the data container & backup container:
    docker rm rancher-data-data-10-25-2023
    docker rm busybox-backup-v2.6.13

6.Restart the Rancher-server container:
    docker start rancher-server-name
       
# Lets say now you need to utilize the backup & restore it your container haaving a issue.

1. Stop the container running Rancher:
   docker stop <RANCHER_CONTAINER_NAME>
   
2. To delete your current state data and replace it with your backup data
docker run  --volumes-from <RANCHER_CONTAINER_NAME> -v $PWD:/backup \
busybox sh -c "rm /var/lib/rancher/* -rf  && \
tar pzxvf /backup/rancher-data-backup-<RANCHER_VERSION>-<DATE>.tar.gz"

3. Restart your Rancher Server container
docker start <RANCHER_CONTAINER_NAME>



















