# Trend Micro Cloud Network Protection

Creates a [Cloud Network Protection](https://resources.trendmicro.com/rs/945-CXD-062/images/DS01_Cloud_Network_Protection_TippingPoint_190710US.pdf) environment in AWS. This demo takes advantage of AWS' newly released [VPC Ingress Routing](https://blog.trendmicro.com/network-security-simplified/) feature.

## Instructions

1. Set the following environment variables:

```
export AWS_REGION=<REGION_NAME>
export STACK_NAME=<STACK_NAME>
```

2. Spin up the environment:

```
aws cloudformation create-stack \
--stack-name $STACK_NAME \
--template-body file://cfn.yml \
--parameters \
ParameterKey=Ec2KeyName,ParameterValue=<KEY_NAME> \
ParameterKey=Ec2AmiId,ParameterValue=<AMI_ID> \
ParameterKey=SmsAmiId,ParameterValue=<AMI_ID> \
ParameterKey=CnpAmiId,ParameterValue=<AMI_ID> \
--capabilities CAPABILITY_IAM
```

3. At the time of writing, it's not possible to set up VPC Ingress Routing using CloudFormation. Wait for the CloudFormation creation to finish running, then execute the following command:

```
aws ec2 associate-route-table --region $AWS_REGION \
--route-table-id $(aws cloudformation --region $AWS_REGION describe-stacks --stack-name $STACK_NAME --query "Stacks[0].Outputs[?OutputKey=='IgwRouteTableId'].OutputValue" --output text) \
--gateway-id $(aws cloudformation --region $AWS_REGION describe-stacks --stack-name $STACK_NAME --query "Stacks[0].Outputs[?OutputKey=='IgwId'].OutputValue" --output text)
``` 

4. Obtain the address of the SMS:

```
aws cloudformation \
--region $AWS_REGION describe-stacks \
--stack-name $STACK_NAME \
--query "Stacks[0].Outputs[?OutputKey=='SmsManagementAddress'].OutputValue" \
--output text
```

5. Install the SMS client. Once installed, log in with the following credentials:

* SMS Username: `SuperUser`
* Password: `Y2VxJuoz!97k#L`

**Note**: The SMS may take 5+ minutes to start. If you cannot connect, wait a moment and then try again.

6. Obtain the address of the CNP:

```
aws cloudformation \
--region $AWS_REGION describe-stacks \
--stack-name $STACK_NAME \
--query "Stacks[0].Outputs[?OutputKey=='CnpManagementAddress'].OutputValue" \
--output text
```

7. SSH into the CNP (`ssh admin@<CNP_MANAGEMENT_ADDRESS> -i <PATH_TO_SSH_KEY>`) and issue the following commands:

```
edit
virtual-segments
virtual-segment "cloud formation"
move to position 1
ips-profile "Default IPS Profile"
reputation-profile "Default Reputation Profile"
address 10.0.0.100/24 10.0.1.100/24
route 0.0.0.0/0 10.0.0.1
bind in-port 1A out-port 1B
bind in-port 1B out-port 1A
exit
commit
exit
high-availability
cloudwatch-health period 1
commit
exit
exit
save-config -y
sms register <SMS_API_KEY> 10.0.2.9 threatdv throughput 1000
```

## Usage

When you apply a profile to the CNP, it will affect the `PublicHost`. To retrieve the `PublicHost`'s IP address, issue the following command:

```
aws cloudformation \
--region $AWS_REGION describe-stacks \
--stack-name $STACK_NAME \
--query "Stacks[0].Outputs[?OutputKey=='PublicHostAddress'].OutputValue" \
--output text
```

## Notes/Troubleshooting

* The lab takes approximately 10 minutes to spin up completely. If you have issues logging into the CNP, SMS or the CNP is not showing up in the SMS, wait a few minutes before trying again.
* If the CNP's system health is `critical`, click the "Refresh" button to retrieve the latest status.

# Contact

* Blog: oznetnerd.com
* Email: will@oznetnerd.com