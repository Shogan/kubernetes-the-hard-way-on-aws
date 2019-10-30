# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network will be setup to host the Kubernetes cluster.

Make sure you have configured your default region. E.g. for eu-west-2:

```
aws configure set default.region eu-west-2
```

Create the `kubernetes-the-hard-way` custom VPC network:

```
aws ec2 create-vpc --cidr-block 10.240.0.0/24
```

Take note of the VpcId of the newly created VPC. E.g.

{
    "Vpc": {
        "VpcId": "vpc-09ad9421856ac72a3", 
        ...
    }
}

### Public Subnet Creation

A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network using the VpcId of your new VPC:

```
aws ec2 create-subnet --vpc-id vpc-09ad9421856ac72a3 --cidr-block 10.240.0.0/24
```

> The `10.240.0.0/24` IP address range can host around 250 compute instances.

Create an Internet Gateway in your VPC to allow instances in the new subnet to talk out to the Internet.

```
aws ec2 create-internet-gateway
```

Take note of the InternetGatewayId. E.g.

```
{
    "InternetGateway": {
        "Attachments": [],
        "InternetGatewayId": "igw-02bb9af57ab048ced",
        "Tags": []
    }
}
```

Attach the new Internet Gateway using the Id to your VPC.

```
aws ec2 attach-internet-gateway --vpc-id vpc-09ad9421856ac72a3 --internet-gateway-id igw-02bb9af57ab048ced
```

Now create a custom route table for the new VPC.

```
aws ec2 create-route-table --vpc-id vpc-09ad9421856ac72a3
```

Make a note of the route table's Id. E.g.

```
{
    "RouteTable": {
        ... 
        "RouteTableId": "rtb-01943b17dfbef2f6b", 
        ...
    }
}
```

Create a route for the new subnet that points traffic destined for the internet (0.0.0.0/0) to your new Internet Gateway.

```
aws ec2 create-route --route-table-id rtb-01943b17dfbef2f6b --destination-cidr-block 0.0.0.0/0 --gateway-id igw-02bb9af57ab048ced
```

Associate the new route table with the new subnet.

Get your new subnet's Id, and use it with the associate-route-table command.

```
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-09ad9421856ac72a3" --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock}'
```

Output example:

```
[
    {
        "ID": "subnet-0c301e597f5640660",
        "CIDR": "10.240.0.0/24"
    }
]
```

Associate it:

```
aws ec2 associate-route-table --subnet-id subnet-0c301e597f5640660 --route-table-id rtb-01943b17dfbef2f6b
```

Finally, set your subnet to automatically assign public IP addresses to instances launched into it.

```
aws ec2 modify-subnet-attribute --subnet-id subnet-0c301e597f5640660 --map-public-ip-on-launch
```

### Firewall Rules

Create a firewall rule that allows external SSH, ICMP, and HTTPS from external, as well as internal communication allowed for all protocols only on the 10.240.0.0/24 and 10.200.0.0/16 CIDRs.

```
aws ec2 create-security-group --group-name kubernetes-the-hard-way --description "kubernetes-the-hard-way" --vpc-id vpc-09ad9421856ac72a3
```

Note the new Security Group Id and use it for the following:

```
aws ec2 authorize-security-group-ingress \
    --group-id sg-023d0a26f29ddb360 \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id sg-023d0a26f29ddb360 \
    --protocol tcp \
    --port 6443 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id sg-023d0a26f29ddb360 \
    --protocol icmp \
    --port -1 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id sg-023d0a26f29ddb360 \
    --protocol all \
    --cidr 10.240.0.0/24

aws ec2 authorize-security-group-ingress \
    --group-id sg-023d0a26f29ddb360 \
    --protocol all \
    --cidr 10.200.0.0/16
```

Note that the above ingress rules allow these ports/protocols the entire internet. Lock them down accordingly if you want to be extra secure, but for the purpose of this lab setup, they are left as open to 0.0.0.0/0.

### Kubernetes Public API access

You'll create an NLB (Network Load Balancer) to provide API access from the internet for remote clients. Note replace the subnet and VPC IDs below with the ones you created earlier.

```
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
    --name kubernetes-the-hard-way \
    --subnets subnet-0c301e597f5640660 \
    --scheme internet-facing \
    --type network \
    --output text --query 'LoadBalancers[].LoadBalancerArn')
  TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
    --name kubernetes-the-hard-way \
    --protocol TCP \
    --port 6443 \
    --vpc-id vpc-09ad9421856ac72a3 \
    --target-type ip \
    --output text --query 'TargetGroups[].TargetGroupArn')
  aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets Id=10.240.0.1{0,1,2}
  aws elbv2 create-listener \
    --load-balancer-arn ${LOAD_BALANCER_ARN} \
    --protocol TCP \
    --port 6443 \
    --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
    --output text --query 'Listeners[].ListenerArn'
```

Note: the target IDs are preallocated IP addresses which will shortly be used to create the Kubernetes controller instances.

Get the public DNS name for the Network Load Balancer, which will be later on used when generating certificates:

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
```

Another note:

Later on, once up and running you may notice error logs on your controller nodes from the `kube-apiserver` along the lines of:

```
http: TLS handshake error from x.x.x.x
```

These are likely related to the NLB's use of a TCP health check as opposed to SSL.

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

Create a key pair to start with. Use the `--query` and `--output` options to pipe the private key directly into a file with .pem extension.

```
aws ec2 create-key-pair --key-name kubernetes-the-hard-way --query 'KeyMaterial' --output text > kubernetes-the-hard-way.pem
chmod 400 kubernetes-the-hard-way.pem
```

Note, before you start the next section, find out the Amazon AMI ID of the Ubuntu Server 18.04 image you will use. These can differ based on what region you are in. For example at the time of writing, `eu-west-2` has `ami-14fb1073`.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane. Note these will be t2.micro to fit in with AWS free tier usage wherever possible.

We'll also disable source/destination IP checking and set some custom user data for use later on when configuring the instances.

The controller instances will have IPs from 10.240.0.10-12

```
for i in 0 1 2; do
  NUM=${i}
  INSTANCE_ID=$(aws ec2 run-instances \
    --image-id ami-14fb1073 \
    --block-device-mapping DeviceName=/dev/sda1,Ebs={VolumeSize=60} \
    --count 1 --instance-type t2.micro \
    --key-name kubernetes-the-hard-way \
    --security-group-ids sg-023d0a26f29ddb360 \
    --subnet-id subnet-0c301e597f5640660 \
    --user-data "name=controller-${NUM}" \
    --private-ip-address 10.240.0.1${i} \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=controller-${NUM}},{Key=Kubernetes-Type,Value=controller}]" \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${INSTANCE_ID} --no-source-dest-check
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports around 254 usable subnets.

The worker instances will have IPs from 10.240.0.20-22. Each will have some metadata set to later on define a bespoke /24 subnet for pods to run on the overlay network.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  NUM=${i}
  INSTANCE_ID=$(aws ec2 run-instances \
    --image-id ami-14fb1073 \
    --block-device-mapping DeviceName=/dev/sda1,Ebs={VolumeSize=60} \
    --count 1 --instance-type t2.micro \
    --key-name kubernetes-the-hard-way \
    --security-group-ids sg-023d0a26f29ddb360 \
    --subnet-id subnet-0c301e597f5640660 \
    --user-data "pod-cidr=10.200.${NUM}.0/24" \
    --private-ip-address 10.240.0.2${i} \
    --user-data "name=worker-${NUM}|pod-cidr=10.200.${NUM}.0/24" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=worker-${NUM}},{Key=Pod-CIDR,Value=10.0.${NUM}.0/24},{Key=Kubernetes-Type,Value=worker}]" \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${INSTANCE_ID} --no-source-dest-check
done
```

### Verification

List the controller and worker instances in your region, providing InstanceId and PublicIpAddress properties:

(Replace the `*controller*` filter in the example below with `*worker*` to view the worker instances too).

```
aws ec2 describe-instances --filters "Name=tag:Name,Values=*controller*" | jq '[.Reservations | .[] | .Instances | .[] | select(.State.Name!="terminated") | {Name: (.Tags[]|select(.Key=="Name")|.Value), InstanceId: .InstanceId, PublicIp: .PublicIpAddress}]'
```

> example output

```
[
  {
    "Name": "controller-2",
    "InstanceId": "i-047ff074d6f70cf7b",
    "PublicIp": "52.x.x.x"
  },
  {
    "Name": "controller-0",
    "InstanceId": "i-01cad84af3f6502c3",
    "PublicIp": "52.x.x.x"
  },
  {
    "Name": "controller-1",
    "InstanceId": "i-06df6afdc0d0fa0d3",
    "PublicIp": "52.x.x.x"
  }
]
```

## Configuring SSH Access

You'll use your previously generated key for SSH access to the instances. They should each have a public IP address as seen above in the listed output.

Test SSH access to the a couple of instances to verify this works (x.x.x.x is the public IP each instance has been assigned):

```
ssh -i ./kubernetes-the-hard-way.pem ubuntu@x.x.x.x
```

Logout once connection tests are successful.

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
