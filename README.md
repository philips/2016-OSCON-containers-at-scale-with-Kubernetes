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
alias kubectl="kubectl --kubeconfig=kubeconfig"
```

And access the cluster to confirm that two nodes have been registered:

```
kubectl --kubeconfig=kubeconfig get nodes
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
