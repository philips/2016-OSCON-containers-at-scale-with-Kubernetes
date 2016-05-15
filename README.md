# Introduction

We are going to turn on three CoreOS VMs under AWS. Then we will deploy a simple web application on top. Once configured we will do various fire drills of different failures of Kubernetes.

## Pre-Requisites

- An [AWS account](http://aws.amazon.com/) and [AWS cli](https://aws.amazon.com/cli/)
  - An [AWS keypair for us-west-2](https://us-west-2.console.aws.amazon.com/ec2/v2/home?region=us-west-2#KeyPairs:sort=keyName)
- [kube-aws](https://coreos.com/kubernetes/docs/latest/kubernetes-on-aws.html) installed and in your path
- [kubectl 1.2.4](https://coreos.com/kubernetes/docs/latest/configure-kubectl.html) installed and in your path

## Testing Pre-Requistes

**AWS CLI**

Test that we have a keypair that works and is avaiable in our 

```
aws ec2 --region us-west-2 describe-key-pairs
{
    "KeyPairs": [
        {
            "KeyName": "philips",
            "KeyFingerprint": "d5:1d:22:c5:cb:57:c3:8d:25:4b:29:f0:f2:9a:96:c9"
        }
    ]
}
```

**kube-aws**

```
kube-aws --help 
kube-aws version v0.7.0
```

## Initial Cluster Setup

```
kube-aws init --cluster-name=my-cluster-name-aws-# \
--external-dns-name=my-cluster-endpoint-aws-# \
--region=us-west-2 \
--availability-zone=us-west-2c \
--key-name=key-pair-name \
--kms-key-arn="arn:aws:kms:us-west-2:334544467761:key/61af0d8f-c27a-466f-a981-81c6ceca6032"
```

Render your cluster assets and initialize the TLS infrastructure needed to securely operate Kubernetes

```
kube-aws render
```

Validate your cluster assets

```
kube-aws validate
```

NOTE: [TBD]


Now lets startup the hosts

```
kube-aws up
```

This will take some time so lets walk through the TLS setup.

### Understanding the Credential Setup

While that is running lets look through the assests that are generated:

```
ls -la credentials/
-rw-------  1 philips  staff  1675 May 13 17:50 admin-key.pem
-rw-------  1 philips  staff  1086 May 13 17:50 admin.pem
-rw-------  1 philips  staff  1675 May 13 17:50 apiserver-key.pem
-rw-------  1 philips  staff  1273 May 13 17:50 apiserver.pem
-rw-------  1 philips  staff  1675 May 13 17:50 ca-key.pem
-rw-------  1 philips  staff  1070 May 13 17:50 ca.pem
-rw-------  1 philips  staff  1679 May 13 17:50 worker-key.pem
-rw-------  1 philips  staff  1155 May 13 17:50 worker.pem
```

From a high level we have generated a CA that is used to sign three different types of keys:

**api server key**: This is the Kubernetes API server certificate to secure API traffic.
**admin key**: This key authenticates kubectl to the API server. It is referenced in kubeconfg.
**worker**: This key authenticates the worker machines to the API server.

Confirm the admin key is signed by the CA by using the openssl tool:

```
openssl x509 -in credentials/admin.pem -text -noout
```

### Configure and Test the Cluster

`kube-aws` can confirm that the cluster is up and running with the `status` sub command:

```
kube-aws status
```

Configure kubectl to use the configuration that kube-aws generated:

```
alias kubectl="kubectl --kubeconfig=${PWD}/kubeconfig"
```

And access the cluster to confirm that two nodes have been registered:

```
kubectl get nodes
```

# Launch Our Production Application

## Application Monitoring

```
kubectl create -f https://raw.githubusercontent.com/kubernetes/contrib/master/prometheus/prometheus-all.json
kubectl create -f https://raw.githubusercontent.com/kubernetes/contrib/master/prometheus/prometheus-service.json
```

# Fire Drills

We are going to run through a series of firedrills now:

- Simulate an API server failure
- Simulate failure of scheduler
- Simulate failure of controller manager
- Recover etcd from a backup
- Move etcd off cluster and scale to 3 machines
- Upgrade of a worker machine
- Failure of a worker machine
- Downgrade/upgrade the kubelet

# Move etcd off cluster and scale to 3 machines

In AWS `kube-aws` sets etcd up on a single machine with an EBS backing store. We have found that this architecture gives reasonable performance and is easy to operate and it is our recommended setup.

But, imagine for that you find you need additional scale from etcd or are designing Kubernetes for availability even in the face of individual VM failure or upgrades.

First, launch an etcd cluster with a [CoreOS stack](https://coreos.com/os/docs/latest/booting-on-ec2.html).

Now lets make some sort of change to the cluster:

```
kubectl scale rc guestbook --replicas=2
```
Launch with userdata

```
#cloud-config

coreos:
  etcd2:
    # generate a new token for each unique cluster from https://discovery.etcd.io/new?size=3
    # specify the initial size of your cluster with ?size=X
    discovery: https://discovery.etcd.io/<token>
    # multi-region and multi-cloud deployments need to use $public_ipv4
    advertise-client-urls: http://$private_ipv4:2379,http://$private_ipv4:4001
    initial-advertise-peer-urls: http://$private_ipv4:2380
    # listen on both the official ports and the legacy ports
    # legacy ports can be omitted if your application doesn't depend on them
    listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    listen-peer-urls: http://$private_ipv4:2380,http://$private_ipv4:7001
```

Get the list of instances for your worker machines

```
aws ec2 describe-instances --filter "Name=tag:aws:cloudformation:stack-name,Values=etcd-cluster" --output text | grep INSTANCES | awk '{print $14 }' > instance-list; cat instance-list
```

Place the backup on the new host

```
first=$(head -n 1 < instance-list )
ssh -A core@k8s.ifup.org  "sudo etcdctl backup --data-dir /var/lib/etcd2 --backup-dir backup; sudo cp -R /var/lib/etcd2/member/wal/ backup/member/; sudo chmod -R 777 backup; scp  -r  -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no backup $first:"
```

```
ssh $first
sudo su
rm -Rf /var/lib/etcd2/
cp -Ra /home/core/backup/ /var/lib/etcd2
chown -R etcd: /var/lib/etcd2
source /etc/environment
machine=$(cat /etc/machine-id)
echo -e "[Service]\nEnvironment=ETCD_FORCE_NEW_CLUSTER=1 ETCD_INITIAL_CLUSTER=${machine}=http://${COREOS_PRIVATE_IPV4}:2380" > /run/systemd/system/etcd2.service.d/90-new-cluster.conf
sudo systemctl daemon-reload
sudo systemctl restart etcd2
```

```
sudo rm /run/systemd/system/etcd2.service.d/90-new-cluster.conf
sudo systemctl daemon-reload
```

```
echo etcdctl member add $(cat /etc/machine-id) http://$(ip addr | grep "inet 10" | awk '{print $2}' | awk -F '/' '{print $1}'):2380
```

```
```

On first run this command

```
etcdctl member add ... > member
echo
echo '[Service]'
cat member | grep ETCD_ | awk '{gsub(/\"/, ""); printf "Environment=%s\n", $1}'
```

Create the systemd unit based on this

```
ETCD_NAME="c2124f7e340f40a882ec163f446bbe80"
ETCD_INITIAL_CLUSTER="c2124f7e340f40a882ec163f446bbe80=http://10.30.19.183:2380,9caf733599824fbc902fdf8deb75f934=http://10.17.15.123:2380,31eb39a0f15f43a5915daaa2767e41d8=http://10.83.4.180:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

- generate systemd stub


- put the updated configs on the machine


- get the cluster running

```
```

- add an elb
- update the running cluster dns to point at new etcd
- restart the API server
- success
