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



## Restoration steps in New K3s Cluster: 

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


## Backing up the Rancher

   # 1. Backing up the present Rancher(Not used to migrate): 
  https://ranchermanager.docs.rancher.com/v2.5/pages-for-subheaders/backup-restore-and-disaster-recovery#changes-in-rancher-v25
  
  # Note: This backup can not be used to migrate your SND install to a Kubernetes cluster.
  
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
  
  # Important: Copy the backup from your node to your local machine or secure it elsewhere. 
    scp & rsync; both tools are capable of transferring files over SSH.
    rsync -P believer@192.168.1.221:/home/believer/rancher-backup-2021-05-28.tar.gz .
    This will copy the backup from the home directory of my server to the current directory my terminal is pointed at.


  # 2. Creating a Persistent Volume for BRO: 
       BRO uses object store to store and retrieve backups.

       1. Create a Persistent Volume for BRO to store the backups.
  
          1. Go to your cluster overview and click on the local cluster. Our local is the cluster that Rancher uses under the hood and it itself is installed in.
          2. Navigate to Storage → Persistent Volumes
   # 3. Click on add volume, and give it the name rancher-backup.For the volume plugin select HostPath, and the default capacity of 10 GB is fine (it’ll be a couple of               MB only). For the path on the node select /rancher-backup, and you must select the option  A directory, or create if does not exist.

  ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/7590ef9e-900a-48c8-a6be-833ead1a4912)


Important: Because Rancher as a single-node docker container runs a K3S single-node cluster inside the Docker container, the path on the node means something different here than you’d expect from the name. The node in this context is not your host system, but rather the container in which Rancher itself is running This means that right now, anything BRO would write to it, will be lost once the container gets removed. Because of this, we’ll have to copy the backup out of the docker container later on in order to use it for the migration.

(Yes, alternatively you could bind-mount a folder on the host to the directory in this container. That way it’ll write directly to the hard disk of your node; feel free to do so.)

  # 3. Installing backup-restore-operator: 
    1. Go to your local cluster. >> Apps & Marketplace.
       Locate the Rancher backups chart and open it.
       
       ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/973c3b14-63d9-4dd8-b9de-b7f32cdfb817)

    2. Go to Chart options,If you are not using S3 object storage, select the “use an existing persistent volume” option and select our previously created rancher-backup PV         as follows.

       ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/c20d7dde-6451-4e2b-9e86-45aea6ce4050)

    3. Click install and wait for the installation to finish. It’ll install two things: first the rancher-backup-crds and secondly rancher-backups. 
       CRDs are custom resource definitions, which are extensions to the Kubernetes API. Rancher backup uses these to create the backups and then restore them.

       ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/0e829d29-1e93-408a-9fe9-12ca7a3745c8)
       
       ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/cf0d4db9-744d-4f49-b645-f0860dc3f3fa)


  # 4. Creating a one-time backup using BRO:
       After we’ve installed BRO, it’s time to create our backup. This is the backup we’ll be using to actually migrate the cluster in the following steps:

       1. Navigate to the cluster explorer of your local cluster like we did before and open the menu on the top left. You’ll notice it has a new option called Rancher       
          backups.
          ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/5e5bc23d-b41d-4bcf-b7b8-77d432d0bea5)

       2. let’s go ahead and make one using the create button.
          ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/8e89f77f-3251-4999-b10d-b9bcddacbef6)

       3. Let’s give our backup a name, called rancher-migrate. Set the schedule to One-Time Backup and set encryption to Store the contents of the backup unencrypted
          (we’ll make it easy on us for ourselves here). 
  # You can leave the storage location set to Use the default storage location configured during installation.
          ![image](https://github.com/believerHSP/HA_K3s_Cluster_setup/assets/101576376/e6c2dfc8-e0a3-45a2-ace5-11a3767d1040)

       4. After you hit create, you should be returning to the backup overview. Your Rancher migrate backup job should be visible and after a few moments the state should              change to completed and a filename should show up.
          Important: Copy the filename to a safe location, you’ll need it later.

 # 5. Extracting the BRO backup from the container for migration:
 
# If you’ve used the persistent volume method from this guide to perform your backup, you’ll have to perform this step. If you’ve used an S3 bucket, you can skip this entirely. If you used a different StorageClass, go download your backup from there.
     1. As mentioned earlier, Rancher uses a K3S cluster inside its docker container in order to run, causing the HostPath PV not to write to the host,
        but to the storage inside the container. This means we’ll have to extract the backup from the container, in order to use it for our migration.

     2. Use the following command to copy the backup from inside the container to a directory on your host:
        sudo docker cp <container-name>:<path_from_pv>/filename_of_backup.tar.gz .

          verify using ls -lrth in host.

     3. This step is also required if you’ll be using a new cluster rather than your current host for your new Rancher installation. Do so by exiting your current host and 
        retrieving the file with rsync.
      rsync -P vashiru@192.168.1.221:/home/vashiru/rancher-migrate-36b16a2b-7f44-4cb1-8a0e-370c3514f681-2021-05-28T08-50-29Z.tar.gz .

     Note: Take note of this filename, as you’ll need it later to perform the restore.

  # 6. Shutting down the current Rancher docker container:
       Now that you’ve created a backup of the current rancher installation and secured it to your local machine for the migration, we can shut down the current Rancher 
       installation. Don’t worry, all workloads in downstream clusters will continue to run as normal.

With the backup out of the way, we’re about halfway there. From here on it’s just a matter of setting up a Kubernetes cluster, preparing it for the restore, restoring the backup, and bringing Rancher back online.  

   # 7. Setup a Cluster: 





       


     


       

      
       




















