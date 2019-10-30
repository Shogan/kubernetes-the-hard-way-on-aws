# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

CNI plugins like calico, weave, and flannel usually offload this task for us with more automated Kubernetes installations, however we'll learn more by adding network routes ourselves using the Amazon VPC routes feature. This method will also allow us to visually inspect our routes in the VPC console afterwards.

## Adding routes to the VPC route table

The following will (for each worker node):

* Get the instance private IP address
* Get the Instance ID
* Get the POD CIDR that we set on each worker's custom metadata
* Create a route table entry in the main VPC route table with a destination of the POD CIDR and a target of each relevant worker's ENI (Elastic Network Interface).

Make sure your `ROUTE_TABLE_ID` shell variable is still valid and points to your VPC route table Id. Example:

```
ROUTE_TABLE_ID=rtb-01943b17dfbef2f6b
```

Then do:

```
for instance in worker-0 worker-1 worker-2; do
  INSTANCE_ID_IP="$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]')"
  INSTANCE_ID="$(echo "${INSTANCE_ID_IP}" | cut -f1)"
  INSTANCE_IP="$(echo "${INSTANCE_ID_IP}" | cut -f2)"
  pod_cidr="$(aws ec2 describe-instance-attribute \
    --instance-id "${INSTANCE_ID}" \
    --attribute userData \
    --output text --query 'UserData.Value' \
    | base64 --decode | tr "|" "\n" | grep "^pod-cidr" | cut -d'=' -f2)"
  echo "${INSTANCE_IP} ${pod_cidr}"

  aws ec2 create-route \
    --route-table-id "${ROUTE_TABLE_ID}" \
    --destination-cidr-block "${pod_cidr}" \
    --instance-id "${INSTANCE_ID}"
done
```

## Checking the routes

Use the VPC / Route Tables / Routes console to verify the new routes are showing as `active`.

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
