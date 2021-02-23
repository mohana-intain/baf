BLOCKCHAIN AUTOMATION FRAMEWORK

CORDA DEPLOYMENT IN AMAZON WEB SERVICE - ELASTIC KUBERNETES SERVICE CLUSTER

A. STEPS TO SET UP AWS EKS CLUSTER

  **TO INSTALL AWS CLI EXECUTE THE FOLLOWING COMMANDS
				
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
  
  **TO CONFIGURE AWS CLI
  
    Run  aws configure. Paste ACCESS KEY AND SECRET KEY PROVIDED.
  
  **TO INSTALL TERRAFORM

    curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
    sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
    sudo apt-get update && sudo apt-get install terraform

  **TO CREATE EKS CLUSTER USING TERRAFORM EKS
  
    git clone git@github.com:sivaramkannan/blockchain-platform.git
    cd blockchain-platform/cloudplatform/aws
    gedit aws_credentials.  //  Place ACCESS KEY AND SECRET KEY PROVIDED.
    terraform init
    terraform state replace-provider "registry.terraform.io/-/aws" "hashicorp/aws"
    terraform apply         
     
  /*The above command creates a Elastic Kubernetes Cluster in AWS with 3 Nodes. In order to increase the number of nodes donot change in the eks.tf . Change in the AWS EKS UI after the Cluster is created.*/
    
    terraform destroy     //  Destroys the cluster.
    
  /*How to change the nodes in UI.

Services -> Containers -> Elastic Kubernetes Services ->Under ”Amazon EKS”  Clusters ->Under “Cluster name” <your cluster> -> Configuration -> Compute -> Node Groups -> Under “Group name” <your nodeGroup> -> Edit (in top right corner) -> Enter the total nodes in Maximum and Desired Size -> Save Changs (Bottom Right Corner) */
  
   **TO CHECK WHETHER THE CLUSTER IS CREATED
        
     aws eks list-clusters
     
   **TO UPDATE THE KUBERNETES CONFIGURATION FILE
   
     aws eks --region <cluster_region> update-kubeconfig --name <cluster_name>
     kubectl config view   
     
   **TO VIEW ALL PODS IN THE CLUSTER
   
     kubectl get pods --all-namespaces
     
B. CREATE A NGINX SERVER

   **CREATE A FILE nginx.yaml AND PASTE THE BELOW CONTENTS
      
        apiVersion: apps/v1
        kind: Deployment
        metadata:
         name: nginx-deployment
        spec:
         replicas: 1
         selector:
             matchLabels:
               app: node-nginx-app
         template:
           metadata:
             labels:
               app: node-nginx-app 
           spec:     
             volumes:
             - name: nginx
               persistentVolumeClaim:
                 claimName: nginx-pvc
             containers:
             - name: node-nginx-app
               image: nginx
               imagePullPolicy: Always
               volumeMounts:
               - mountPath: /mnt
                 name: nginx
                 
   **TO DEPLOY THE SERVER, EXECUTE THE BELOW COMMANDS
   
      kubectl create deployment nginx --image=nginx
      kubectl get pods --all-namespaces
      kubectl exec -it <nginx pod name> -- /bin/bash

C. SETTING UP VAULT SERVER

    **AFTER LOGGING INTO CONSOLE CHOOSE Services → Compute → EC2 → Launch Instance → Launch Instance 

    **SELECT ami-0a91cd140a1fc148a / ami-0dd9f0e7df0f0a138 / ami-0d2751e39abf67ea8 / ami-0ebc8f6f580a04647

    **TYPE:  t2.micro

    **ADD RULE :
      ingress {
         from_port = 8200        
         to_port = 8200
         protocol = "tcp"
         cidr_blocks = ["0.0.0.0/0"]
         }

      ingress {
         from_port = 8200
         to_port = 8200
         protocol = "tcp"
         ipv6_cidr_blocks = ["::/0"]
      }

    **SELECT YOUR EXISTING PAIR OR CREATE A NEW KEY PAIR

    **LAUNCH Instance
    
    **CLICK How to connect to your Linux Instance
    CHOOSE SSH  Or go to the instance dashboard and click connect
    Copy the command paste in terminal
    Check whether key-pair.pem file exists in the directory before you execute the command

    **AFTER LOGGING INTO THE INSTANCE DO THE FOLLOWING
    
        https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04 
        Follow the link to install docker
        which docker
        cd into directory
        sudo chmod -R 777 docker*
    **DO THE FOLLOWING

1. Create a file /etc/systemd/system/vault-docker.service
    cd  /etc/systemd/system
    nano vault-docker.service
2. Copy paste the below content there

[Unit]

Description=Vault Docker Application Service
Requires=docker.service
After=docker.service

[Service]

ExecStart=/usr/bin/docker run --rm -v /opt/vault/store:/vault/file -p
8200:8200 --cap-add=IPC_LOCK -e 'VAULT_LOCAL_CONFIG={"backend": {"file":
{"path": "/vault/file"}}, "default_lease_ttl": "168h", "max_lease_ttl":
"720h", "listener": {"tcp": {"address": "0.0.0.0:8200", "tls_disable":
true}}}' vault server
Restart=always
RestartSec=90
StartLimitInterval=400
StartLimitBurst=3

[Install]

WantedBy=multi-user.target

3. sudo systemctl enable vault-docker.service
4. sudo systemctl start vault-docker.service

    **EXECUTE THE COMMAND TO VIEW THE IP
        dig +short myip.opendns.com @resolver1.opendns.com

    **GO TO THE LINUX TERMINAL [Use the Above IP to export]

        export VAULT_ADDR='http://DNS_Name:8200' //DROPLET IP
        vault operator init -key-shares=1 -key-threshold=1
        vault login root-token
        export VAULT_TOKEN="<Your Vault root token>"
        vault operator unseal
        vault secrets enable -version=1 -path=secret kv
        vault status
        vault secrets list
