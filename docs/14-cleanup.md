# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
aws ec2 terminate-instances --instance-ids ${worker1} ${worker2} ${worker3} ${controller1} ${controller2} ${controller3}
```

## Networking

Delete the external load balancer network resources:

```
aws elb delete-load-balancer --load-balancer-name kubernetes-api
```

```
aws iam remove-role-from-instance-profile --instance-profile-name hardway-workers-Instance-Profile --role-name hardway-workers
aws iam delete-instance-profile --instance-profile-name hardway-workers-Instance-Profile
aws iam remove-role-from-instance-profile --instance-profile-name hardway-controllers-Instance-Profile --role-name hardway-controllers
aws iam delete-instance-profile --instance-profile-name hardway-controllers-Instance-Profile
aws iam delete-role-policy --role-name hardway-workers --policy-name hardway-workers-Permissions
aws iam delete-role-policy --role-name hardway-controllers --policy-name hardway-controllers-Permissions
aws iam delete-role --role-name hardway-workers
aws iam delete-role --role-name hardway-controllers
aws ec2 delete-security-group --group-id ${$sgid}
aws ec2 disassociate-route-table  --subnet-id ${private_subnet3id} --route-table-id ${private_rtb3id}
aws describe-route-tables

aws describe-route-tables --output text --query 'RouteTables[*].Associations[*].RouteTableAssociationId'
aws ec2 describe-route-tables --output text --query 'RouteTables[*].Associations[*].RouteTableAssociationId'

aws ec2 delete-subnet  --subnet-id $private_subnet3id
aws ec2 delete-route-table --route-table-id $private_rtb3id
aws ec2 delete-subnet  --subnet-id $private_subnet2id
aws ec2 delete-route-table --route-table-id $private_rtb2id
aws ec2 delete-subnet  --subnet-id $private_subnet1id
aws ec2 delete-route-table --route-table-id $private_rtb1id

aws ec2 delete-nat-gateway --nat-gateway-id $natgw3id
aws ec2 delete-nat-gateway --nat-gateway-id $natgw2id
aws ec2 delete-nat-gateway --nat-gateway-id $natgw1id

aws ec2 release-address --allocation-id $eip3id
aws ec2 release-address --allocation-id $eip2id
aws ec2 release-address --allocation-id $eip1id

aws ec2 delete-subnet  --subnet-id $subnet3id
aws ec2 delete-subnet  --subnet-id $subnet2id
aws ec2 delete-subnet  --subnet-id $subnet1id
aws ec2 delete-route-table --route-table-id $rtbid

aws ec2 detach-internet-gateway --internet-gateway-id $igwid --vpc-id $vpcid
aws ec2 delete-internet-gateway  --internet-gateway-id $igwid

aws ec2 delete-vpc --vpc-id $vpcid

```
