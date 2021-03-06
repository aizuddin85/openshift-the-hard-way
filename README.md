#  CAUTION ------WORK IN PROGRES------     

# openshift-the-hard-way

## On all nodes/master instances
```sh
sudo yum install -y centos-release-openshift-origin
sudo yum install -y origin-clients
sudo yum install -y origin
sudo yum install -y docker
#edit /etc/sysconfig/docker file and add --insecure-registry 172.30.0.0/16 to the OPTIONS parameter.
sudo sed -i '/OPTIONS=.*/c\OPTIONS="--selinux-enabled --insecure-registry 172.30.0.0/16"' \
/etc/sysconfig/docker
sudo systemctl is-active docker
sudo systemctl enable docker
sudo systemctl restart docker
```

## On Master let's say 10.128.0.2

```sh
mkdir -p ocp/master

openshift start master --dns='tcp://0.0.0.0:8053' --public-master='https://10.128.0.2:8443' \
--listen='https://0.0.0.0:8443' \
--master='https://10.128.0.2:8443' \
--write-config='ocp/master'
openshift start master --config=ocp/master/master-config.yaml
export KUBECONFIG=$(pwd)/ocp/master/admin.kubeconfig
oc get nodes
scp -r ocp node1:/tmp
```

## On nodes  let's say 10.128.0.3
```sh
oc adm create-node-config \
    --master='https://10.128.0.2:8443' \
    --node-dir=ocp/node1 \
    --node=node1 \
    --hostnames=node1,10.128.0.3 \
    --certificate-authority="ocp/master/ca.crt" \
    --signer-cert="ocp/master/ca.crt" \
    --signer-key="ocp/master/ca.key" \
    --signer-serial="ocp/master/ca.serial.txt" \
    --node-client-certificate-authority="ocp/master/ca.crt"

openshift start node --config=node-config.yaml
export KUBECONFIG=$(pwd)/ocp/master/admin.kubeconfig
oc get nodes 
oc adm policy add-scc-to-user hostnetwork -z router
oc adm router
oc new-app debianmaster/go-welcome
oc expose svc go-welcome --hostname=go-welcome.tmp.xfc.io
```




# Gcloud
```sh
gcloud compute instances list 

gcloud config set compute/region asia-east1

gcloud config set compute/zone asia-east1-a

gcloud compute networks create openshift --mode custom

gcloud compute networks subnets create openshift-subnet \
  --network openshift \
  --range 10.240.0.0/24
  

gcloud compute firewall-rules create allow-internal \
  --allow tcp,udp,icmp \
  --network openshift \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
  
 
gcloud compute firewall-rules create allow-external \
  --allow tcp:22,tcp:3389,tcp:6443,tcp:8443,icmp \
  --network openshift \
  --source-ranges 0.0.0.0/0  
  
  
gcloud compute firewall-rules create allow-healthz \
  --allow tcp:8080 \
  --network openshift \
  --source-ranges 130.211.0.0/22 
  

gcloud compute firewall-rules create allow-http-https   --allow tcp:8080,tcp:443   --network openshift   --source-ranges 0.0.0.0/0

gcloud compute firewall-rules list --filter "network=openshift"

gcloud compute addresses create openshift --region=asia-east1

gcloud compute addresses list openshift

gcloud compute instances create "master1" --zone "asia-east1-a" --machine-type n1-standard-1 \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "master1"  \
  --private-network-ip 10.240.0.10  --subnet openshift-subnet
  

gcloud compute instances create "master2" --zone "asia-east1-a" --machine-type n1-standard-1 \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "master2"  \
  --private-network-ip 10.240.0.11  --subnet openshift-subnet
  
  
gcloud compute instances create "master3" --zone "asia-east1-a" --machine-type n1-standard-1 \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "master3"  \
  --private-network-ip 10.240.0.12  --subnet openshift-subnet  

 gcloud compute instances create "infra1" --zone "asia-east1-a" --machine-type n1-standard-1  \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "infra1"  \
  --private-network-ip 10.240.0.51  --subnet openshift-subnet   
  

 gcloud compute instances create "infra2" --zone "asia-east1-a" --machine-type n1-standard-1  \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "infra2"  \
  --private-network-ip 10.240.0.52  --subnet openshift-subnet  
  
  
gcloud compute instances create "node1" --zone "asia-east1-a" --machine-type n1-standard-1  \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "node1"  \
  --private-network-ip 10.240.0.75  --subnet openshift-subnet  


> Copy cat ~/.ssh/id_rsa.pub   to metadata
gcloud compute copy-files ~/.ssh/id_rsa master1:~/
gcloud compute ssh master1
sudo yum install -y centos-release-openshift-origin
sudo yum install -y origin-clients
sudo yum install -y origin
sudo yum --enablerepo=centos-openshift-origin-testing install atomic-openshift-utils 
ssh-agent $SHELL
ssh-add ~/id_rsa
``` 


`vi /usr/lib/systemd/system/xfc-master-api.service`
```sh
[Unit]
Description=API Server

[Service]
Type=notify
WorkingDirectory=/opt/ocp
ExecStart=/opt/ocp/openshift start master api --config=master/master-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
`vi /usr/lib/systemd/system/xfc-master-controller.service` 

```sh
Unit]
Description=API Controllers

[Service]
Type=notify
WorkingDirectory=/opt/ocp
# set GOMAXPROCS to number of processors
ExecStart=/opt/ocp/openshift start master controllers --config=master/master-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

`vi /usr/lib/systemd/system/xfc-node.service`  

```sh
[Unit]
Description=Node

[Service]
Type=notify
WorkingDirectory=/opt/ocp
# set GOMAXPROCS to number of processors
ExecStart=/opt/ocp/openshift start node --config=node1/node-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```



```sh
aws ec2 describe-instances \
  --filters "Name=tag:cluster,Values=chak" | \
  jq -j '.Reservations[].Instances[] | .InstanceId, "  ", .Placement.AvailabilityZone, "  ", .PrivateIpAddress, "  ", .PublicIpAddress, "\n"'
  
aws ec2 describe-instances   --filters "Name=tag:cluster,Values=chak" |   jq -j '.Reservations[].Instances[] | .PrivateIpAddress, "  ", .PublicIpAddress, "\n"'  
  ```


```sh
master_routingconfig_subdomain: apps.ck.osecloud.com
  openshift_master_cluster_hostname: ck.osecloud.com
  openshift_master_cluster_public_hostname: ck.osecloud.com
  openshift_master_api_port: 443
  openshift_master_console_port: 443
  openshift_hosted_manage_router: true
  openshift_hosted_manage_registry: true
  openshift_hosted_router_selector: 'region=infra'
  openshift_hosted_registry_selector: 'region=infra'
  openshift_hosted_metrics_deploy: true
  openshift_master_logging_public_url: https://kibana.apps.ck.osecloud.com
  openshift_hosted_logging_deploy: true
  openshift_master_identity_providers: [{'name': 'allow_all', 'login': 'true', 'challenge': 'true', 'kind': 'AllowAllPasswordIdentityProvider'}]
  ```


### Additional notes
#### For route53 updates

```sh

export record_name=apps.ck.osecloud.com
export record_value=13.59.37.234
export ttl=60
export action=UPSERT
export record_type=A

expport zone_id=$(aws route53 list-hosted-zones | jq -r ".HostedZones[] | select(.Name == \"ck.osecloud.com.\") | .Id" | cut -d'/' -f3)


function change_batch() {
	jq -c -n "{\"Changes\": [{\"Action\": \"$action\", \"ResourceRecordSet\": {\"Name\": \"$record_name\", \"Type\": \"$record_type\", \"TTL\": $ttl, \"ResourceRecords\": [{\"Value\": \"$record_value\"} ] } } ] }"
}

aws route53 change-resource-record-sets --hosted-zone-id ${zone_id} --change-batch $(change_batch) | jq -r '.ChangeInfo.Id' | cut -d'/' -f3
```
