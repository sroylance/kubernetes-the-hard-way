# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster.

> Ensure the aws command line has been configured as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

Determine what availability zones exist in your region using
```
aws ec2 describe-availability-zones
```
We will assume using us-east-1a, us-east-1b and us-east-1c

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-subnets-commands-example.html) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```
vpcid=`aws ec2 create-vpc --cidr-block 10.251.0.0/16 --query 'Vpc.VpcId' --output text`
aws ec2 create-tags --resources ${vpcid} --tags Key=Name,Value=hardway Key=KubernetesCluster,Value=hardway Key=kubernetes.io/cluster/hardway,Value=owned
aws ec2 modify-vpc-attribute --vpc-id ${vpcid}  --enable-dns-hostnames
```

Subnets must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

We will build a public/private network topology with 6 subnets across 3 availabilty zones

```
igwid=`aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text`
aws ec2 attach-internet-gateway --vpc-id ${vpcid} --internet-gateway-id ${igwid}
rtbid=`aws ec2 create-route-table --vpc-id ${vpcid} --query 'RouteTable.RouteTableId' --output text`
aws ec2 create-route --route-table-id ${rtbid} --destination-cidr-block 0.0.0.0/0 --gateway-id ${igwid}
aws ec2 create-tags --resources ${rtbid} --tags Key=KubernetesCluster,Value=hardway
```

```
subnet1id=`aws ec2 create-subnet --vpc-id ${vpcid} --cidr-block 10.251.12.0/22 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output=text`
aws ec2 create-tags --resources ${subnet1id} --tags Key=Name,Value=public1a Key=KubernetesCluster,Value=hardway Key=kubernetes.io/cluster/hardway,Value=owned Key=kubernetes.io/role/elb,Value=1
aws ec2 associate-route-table  --subnet-id ${subnet1id} --route-table-id ${rtbid}
aws ec2 modify-subnet-attribute --subnet-id ${subnet1id} --map-public-ip-on-launch

subnet2id=`aws ec2 create-subnet --vpc-id ${vpcid} --cidr-block 10.251.16.0/22 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output=text`
aws ec2 create-tags --resources ${subnet2id} --tags Key=Name,Value=public1b Key=KubernetesCluster,Value=hardway Key=kubernetes.io/cluster/hardway,Value=owned Key=kubernetes.io/role/elb,Value=1
aws ec2 associate-route-table  --subnet-id ${subnet2id} --route-table-id ${rtbid}
aws ec2 modify-subnet-attribute --subnet-id ${subnet2id} --map-public-ip-on-launch

subnet3id=`aws ec2 create-subnet --vpc-id ${vpcid} --cidr-block 10.251.20.0/22 --availability-zone us-east-1c --query 'Subnet.SubnetId' --output=text`
aws ec2 create-tags --resources ${subnet3id} --tags Key=Name,Value=public1c Key=KubernetesCluster,Value=hardway Key=kubernetes.io/cluster/hardway,Value=owned Key=kubernetes.io/role/elb,Value=1
aws ec2 associate-route-table  --subnet-id ${subnet3id} --route-table-id ${rtbid}
aws ec2 modify-subnet-attribute --subnet-id ${subnet1id} --map-public-ip-on-launch
```

```
eip1id=`aws ec2 allocate-address --query 'AllocationId' --output=text`
natgw1id=` aws ec2 create-nat-gateway --subnet-id ${subnet1id} --allocation-id ${eip1id} --query 'NatGateway.NatGatewayId' --output=text`

eip2id=`aws ec2 allocate-address --query 'AllocationId' --output=text`
natgw2id=` aws ec2 create-nat-gateway --subnet-id ${subnet2id} --allocation-id ${eip2id} --query 'NatGateway.NatGatewayId' --output=text`

eip3id=`aws ec2 allocate-address --query 'AllocationId' --output=text`
natgw3id=` aws ec2 create-nat-gateway --subnet-id ${subnet3id} --allocation-id ${eip3id} --query 'NatGateway.NatGatewayId' --output=text`
```
```
private_subnet1id=`aws ec2 create-subnet --vpc-id ${vpcid} --cidr-block 10.251.0.0/22 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output=text`
aws ec2 create-tags --resources ${private_subnet1id} --tags Key=Name,Value=private1a Key=KubernetesCluster,Value=hardway Key=kubernetes.io/cluster/hardway,Value=owned Key=kubernetes.io/role/elb,Value=1
private_rtb1id=`aws ec2 create-route-table --vpc-id ${vpcid} --query 'RouteTable.RouteTableId' --output text`
aws ec2 create-tags --resources ${private_rtb1id} --tags Key=KubernetesCluster,Value=hardway
aws ec2 create-route --route-table-id ${private_rtb1id} --destination-cidr-block 0.0.0.0/0 --nat-gateway-id ${natgw1id}
aws ec2 associate-route-table  --subnet-id ${private_subnet1id} --route-table-id ${private_rtb1id}


private_subnet2id=`aws ec2 create-subnet --vpc-id ${vpcid} --cidr-block 10.251.4.0/22 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output=text`
aws ec2 create-tags --resources ${private_subnet2id} --tags Key=Name,Value=private1b Key=KubernetesCluster,Value=hardway Key=kubernetes.io/cluster/hardway,Value=owned Key=kubernetes.io/role/elb,Value=1
private_rtb2id=`aws ec2 create-route-table --vpc-id ${vpcid} --query 'RouteTable.RouteTableId' --output text`
aws ec2 create-tags --resources ${private_rtb2id} --tags Key=KubernetesCluster,Value=hardway
aws ec2 create-route --route-table-id ${private_rtb2id} --destination-cidr-block 0.0.0.0/0 --nat-gateway-id ${natgw2id}
aws ec2 associate-route-table  --subnet-id ${private_subnet2id} --route-table-id ${private_rtb2id}


private_subnet3id=`aws ec2 create-subnet --vpc-id ${vpcid} --cidr-block 10.251.8.0/22 --availability-zone us-east-1c --query 'Subnet.SubnetId' --output=text`
aws ec2 create-tags --resources ${private_subnet3id} --tags Key=Name,Value=private1c Key=KubernetesCluster,Value=hardway Key=kubernetes.io/cluster/hardway,Value=owned Key=kubernetes.io/role/elb,Value=1
private_rtb3id=`aws ec2 create-route-table --vpc-id ${vpcid} --query 'RouteTable.RouteTableId' --output text`
aws ec2 create-tags --resources ${private_rtb3id} --tags Key=KubernetesCluster,Value=hardway
aws ec2 create-route --route-table-id ${private_rtb3id} --destination-cidr-block 0.0.0.0/0 --nat-gateway-id ${natgw3id}
aws ec2 associate-route-table  --subnet-id ${private_subnet3id} --route-table-id ${private_rtb3id}
```


```
cat > hardway.env <<EOF
vpcid=${vpcid}
igwid=${igwid}
rtbid=${rtbid}
subnet1id=${subnet1id}
subnet2id=${subnet2id}
subnet3id=${subnet3id}
eip1id=${eip1id}
eip2id=${eip2id}
eip3id=${eip3id}
natgw1id=${natgw1id}
natgw2id=${natgw2id}
natgw3id=${natgw3id}
private_subnet1id=${private_subnet1id}
private_subnet2id=${private_subnet2id}
private_subnet3id=${private_subnet3id}
private_rtb1id=${private_rtb1id}
private_rtb2id=${private_rtb2id}
private_rtb3id=${private_rtb3id}
EOF
```

### Firewall Rules

Create a firewall rule that allows internal communication across all protocols:

```
sgid=`aws ec2 create-security-group --vpc-id ${vpcid} --group-name hardway  --description "Allow everything from everywhere.  NEVER USE THIS FOR REAL" --query 'GroupId' --output=text`
aws ec2 authorize-security-group-ingress  --group-id ${sgid} --protocol all  --cidr 0.0.0.0/0
```

### External access
at this point the private kubernetes network is completely inaccesible from the outside.  You will need to create a mechanism that allows you to communicate with it such as a bastion host, VPC peering or a VPN connection.

## Compute Instances

The compute instances in this lab will be provisioned using [Debian](https://wiki.debian.org/Cloud/AmazonEC2Image/Stretch) 9. Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.


### IAM instance profiles
https://github.com/kubernetes/kops/blob/master/docs/iam_roles.md
https://docs.aws.amazon.com/codedeploy/latest/userguide/getting-started-create-iam-instance-profile.html

```
cat > EC2-Trust.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF
```

```
cat > policy-workers.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "hardwayK8sEC2NodePerms",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeRegions"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF
```

```
cat > policy-controllers.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "hardwayK8sEC2MasterPermsDescribeResources",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeRouteTables",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeVolumes"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "hardwayK8sEC2MasterPermsAllResources",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSecurityGroup",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:ModifyInstanceAttribute"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "hardwayK8sEC2MasterPermsTaggedResources",
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CreateRoute",
        "ec2:DeleteRoute",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteVolume",
        "ec2:DetachVolume",
        "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "hardwayK8sASMasterPermsAllResources",
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:GetAsgForInstance"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "hardwayK8sASMasterPermsTaggedResources",
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "autoscaling:UpdateAutoScalingGroup"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "hardwayK8sELBMasterPermsRestrictive",
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:AddTags",
        "elasticloadbalancing:AttachLoadBalancerToSubnets",
        "elasticloadbalancing:ApplySecurityGroupsToLoadBalancer",
        "elasticloadbalancing:CreateLoadBalancer",
        "elasticloadbalancing:CreateLoadBalancerPolicy",
        "elasticloadbalancing:CreateLoadBalancerListeners",
        "elasticloadbalancing:ConfigureHealthCheck",
        "elasticloadbalancing:DeleteLoadBalancer",
        "elasticloadbalancing:DeleteLoadBalancerListeners",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticloadbalancing:DescribeLoadBalancerAttributes",
        "elasticloadbalancing:DetachLoadBalancerFromSubnets",
        "elasticloadbalancing:DeregisterInstancesFromLoadBalancer",
        "elasticloadbalancing:ModifyLoadBalancerAttributes",
        "elasticloadbalancing:RegisterInstancesWithLoadBalancer",
        "elasticloadbalancing:SetLoadBalancerPoliciesForBackendServer"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "hardwayK8sNLBMasterPermsRestrictive",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "elasticloadbalancing:AddTags",
        "elasticloadbalancing:CreateListener",
        "elasticloadbalancing:CreateTargetGroup",
        "elasticloadbalancing:DeleteListener",
        "elasticloadbalancing:DeleteTargetGroup",
        "elasticloadbalancing:DescribeListeners",
        "elasticloadbalancing:DescribeLoadBalancerPolicies",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeTargetHealth",
        "elasticloadbalancing:ModifyListener",
        "elasticloadbalancing:ModifyTargetGroup",
        "elasticloadbalancing:RegisterTargets",
        "elasticloadbalancing:SetLoadBalancerPoliciesOfListener"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "hardwayMasterCertIAMPerms",
      "Effect": "Allow",
      "Action": [
        "iam:ListServerCertificates",
        "iam:GetServerCertificate"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF
```

```
aws iam create-role --role-name hardway-controllers --assume-role-policy-document file://EC2-Trust.json
aws iam create-role --role-name hardway-workers --assume-role-policy-document file://EC2-Trust.json
aws iam put-role-policy --role-name hardway-controllers --policy-name hardway-controllers-Permissions --policy-document file://policy-controllers.json
aws iam put-role-policy --role-name hardway-workers --policy-name hardway-workers-Permissions --policy-document file://policy-workers.json

aws iam create-instance-profile --instance-profile-name hardway-controllers-Instance-Profile
aws iam add-role-to-instance-profile --instance-profile-name hardway-controllers-Instance-Profile --role-name hardway-controllers

aws iam create-instance-profile --instance-profile-name hardway-workers-Instance-Profile
aws iam add-role-to-instance-profile --instance-profile-name hardway-workers-Instance-Profile --role-name hardway-workers
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
ami=ami-71d8820b
controller1=`aws ec2 run-instances --subnet-id ${private_subnet1id} --private-ip-address 10.251.0.10 --image-id ${ami} --key-name hardway --security-group-ids ${sgid} --instance-type t2.small --iam-instance-profile Name=hardway-controllers-Instance-Profile --block-device-mappings '[{"DeviceName":"xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' --query 'Instances[0].InstanceId' --output text`
aws ec2 create-tags --resources $controller1 --tag Key=Name,Value=controller-1 Key=KubernetesCluster,Value=hardway
controller2=`aws ec2 run-instances --subnet-id ${private_subnet2id} --private-ip-address 10.251.4.10 --image-id ${ami} --key-name hardway --security-group-ids ${sgid} --instance-type t2.small --iam-instance-profile Name=hardway-controllers-Instance-Profile --block-device-mappings '[{"DeviceName":"xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' --query 'Instances[0].InstanceId' --output text`
aws ec2 create-tags --resources $controller2 --tag Key=Name,Value=controller-2 Key=KubernetesCluster,Value=hardway
controller3=`aws ec2 run-instances --subnet-id ${private_subnet3id} --private-ip-address 10.251.8.10 --image-id ${ami} --key-name hardway --security-group-ids ${sgid} --instance-type t2.small --iam-instance-profile Name=hardway-controllers-Instance-Profile --block-device-mappings '[{"DeviceName":"xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' --query 'Instances[0].InstanceId' --output text`
aws ec2 create-tags --resources $controller3 --tag Key=Name,Value=controller-3 Key=KubernetesCluster,Value=hardway
```

### Kubernetes Workers

Create three compute instances which will host the Kubernetes worker nodes:

```
worker1=`aws ec2 run-instances --subnet-id ${private_subnet1id} --private-ip-address 10.251.0.20 --image-id ${ami} --key-name hardway --security-group-ids ${sgid} --instance-type t2.small --iam-instance-profile Name=hardway-controllers-Instance-Profile --block-device-mappings '[{"DeviceName":"xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' --query 'Instances[0].InstanceId' --output text`
aws ec2 create-tags --resources $worker1 --tag Key=Name,Value=worker-1 Key=KubernetesCluster,Value=hardway
worker2=`aws ec2 run-instances --subnet-id ${private_subnet2id} --private-ip-address 10.251.4.20 --image-id ${ami} --key-name hardway --security-group-ids ${sgid} --instance-type t2.small --iam-instance-profile Name=hardway-controllers-Instance-Profile --block-device-mappings '[{"DeviceName":"xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' --query 'Instances[0].InstanceId' --output text`
aws ec2 create-tags --resources $worker2 --tag Key=Name,Value=worker-2 Key=KubernetesCluster,Value=hardway
worker3=`aws ec2 run-instances --subnet-id ${private_subnet3id} --private-ip-address 10.251.8.20 --image-id ${ami} --key-name hardway --security-group-ids ${sgid} --instance-type t2.small --iam-instance-profile Name=hardway-controllers-Instance-Profile --block-device-mappings '[{"DeviceName":"xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' --query 'Instances[0].InstanceId' --output text`
aws ec2 create-tags --resources $worker3 --tag Key=Name,Value=worker-3 Key=KubernetesCluster,Value=hardway
```

### trust SSH host keys
ssh-keyscan -H ip-10-251-0-10 ip-10-251-4-10 ip-10-251-8-10 ip-10-251-0-20 ip-10-251-4-20 ip-10-251-8-20 >> ~/.ssh/known_hosts
ssh-keyscan -H 10.251.0.10 10.251.4.10 10.251.8.10 10.251.0.20 10.251.4.20 10.251.8.20>> ~/.ssh/known_hosts


### API load balancers
'''
elbdns=`aws elb create-load-balancer --load-balancer-name kubernetes-api --subnets ${private_subnet1id} ${private_subnet1d} ${private_subnet3id} --scheme internal --listeners "Protocol=TCP,LoadBalancerPort=6443,InstanceProtocol=TCP,InstancePort=6443" --query "DNSName" --output text`
aws elb register-instances-with-load-balancer --load-balancer-name kubernetes-api --instances $controller1 $controller2 $controller3
'''

### Verification

List the compute instances in your default compute zone:

```
aws ec2 describe-instances --output text --query 'Reservations[*].Instances[*].{ID:InstanceId,IP:PrivateIpAddress,State:State.Name}'
```

> output

```
i-0bada7ef6bca59607     10.251.0.20     running
i-0cd8a9164c01b764f     10.251.0.10     running
i-0448b1ef53cd4ad83     10.251.8.10     running
i-05c1b5d64e171adbc     10.251.8.20     running
i-0a466d099f13dea67     10.251.4.20     running
i-0d17594a39a2e86c8     10.251.4.10     running

```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
