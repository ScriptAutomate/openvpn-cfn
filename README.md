# Deploy OpenVPN to AWS via CloudFormation and Amazon Linux 2

This is a new and improved version of a CFN template blogged about by [Linux Academy](https://linuxacademy.com/).

> If you want to see the original template that they worked on, taking note that it is out-dated and no longer functional, you can take a look at it here:
> - [GitHub Gist: vpn-cloudformation-template.yaml](https://gist.github.com/pbzona/13fd0d9d12a7cc7492007ed370b677a0)
>
> The following blog articles walkthrough how the base CFN template was initially created, along with full descriptions as to why each resource is used:
> - [How to Roll Your Own VPN with AWS CloudFormation – Part One](https://linuxacademy.com/blog/tutorials/roll-vpn-aws-cloudformation-part-one/)
> - [How to Roll Your Own VPN with AWS CloudFormation – Part Two](https://linuxacademy.com/blog/tutorials/roll-vpn-aws-cloudformation-part-two/)
> - [How to Roll Your Own VPN with AWS CloudFormation – Part Three](https://linuxacademy.com/blog/tutorials/how-to-roll-your-own-vpn-with-aws-cloudformation-part-three/)

## Updates in This New CFN Template

- Updated for Amazon Linux 2
- Added parameter for specifying OpenVPN version. Default: version `2.4.7` (latest available, tested version from EPEL as of November, 2019)
- Updated to export single OVPN instead of .zip of client configuration
- Replaced AMI `Mappings` sections with `LatestAmiId` parameter dynamically referencing [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) for latest Amazon Linux 2 AMI ID in relative region (by default)
  - A simple lookup is now available to CFN templates, as seen here: [Query for the latest Amazon Linux AMI IDs using AWS Systems Manager Parameter Store](https://aws.amazon.com/blogs/compute/query-for-the-latest-amazon-linux-ami-ids-using-aws-systems-manager-parameter-store/)
- `CustomResource` AWS Lambda migrated over to `python3.7` runtime due to `nodejs6.10` no longer being supported and the `nodejs8.10` runtime approaching EOL: [AWS Lambda: Node.js v8.10 Runtime Approaching End of Life (EOL)](https://dev.to/scriptautomate/aws-lambda-node-js-v8-10-runtime-approaching-eol-end-of-life-gll)
  - Source used for new `CustomResource` running on `python3.7` runtime came from: [custom-resource-s3-bucket-delete](https://github.com/mike-mosher/custom-resource-s3-bucket-delete)
- Updated for Easy-RSA v3.x
- Parameterized OpenVPN port and protocol
- Parameterized `EASYRSA_ALGO` (Default: **rsa**, but can also be **ec**) and `EASYRSA_REQ_CN` (Default: `ChangeMe`) Easy-RSA vars
- Updated OpenVPN client and server config values
  - Replaced `ns-cert-type server` with `remote-cert-tls server` in client config
    - Due to `ns-cert-type` slated for full deprecation by v2.5.x of OpenVPN, and due to mobile clients (Android / iOS) currently erroring out if present
  - Removed `comp-lzo` from server and client configs
    - Due to the [VORACLE security advisory](https://community.openvpn.net/openvpn/wiki/VORACLE) (2018)

## Step-by-Step Deployment

This can be done via the GUI by uploading the `cfn-openvpn.yaml` while creating a stack, in the AWS Console, or it can be done via the CLI. If going through the GUI, an SSH key pair needs to be made first via the EC2 service.

Otherwise, keep following for a CLI approach to the deployment that sets up a Python virtualenv, creates an SSH key pair, and deploys the CFN stack.

### Prerequisites

- Python 3 (tested on Python 3.6 and 3.7)

#### `awscli` Setup

Checkout [awscli](https://aws.amazon.com/cli/) if needed. An optional approach to configuring `awscli` can be seen below.

##### `awscli` Setup: Python VirtualEnv

```bash
python3 -m venv venv
source venv/bin/activate
pip install -U pip setuptools
pip install awscli
```

##### `awscli` Setup: Configure AWS CLI Profile

```bash
# Target aws profile to create
AWS_VPN_PROFILE='testprofile'

# Configure Access Keys
aws configure --profile $AWS_VPN_PROFILE
```

### Create EC2 SSH Key Pair

```bash
# Target aws profile to use
AWS_VPN_PROFILE='testprofile'

# Generate SSH Key Pair; set file permissions (Linux/MacOS)
SSHKEYPAIR='openvpn'
aws ec2 create-key-pair \
  --key-name $SSHKEYPAIR \
  --output text \
  --profile $AWS_VPN_PROFILE \
  --query 'KeyMaterial' > $SSHKEYPAIR.pem
chmod 400 $SSHKEYPAIR.pem
```

### Create OpenVPN via CFN

```bash
aws cloudformation deploy \
    --template-file cfn-openvpn.yaml \
    --stack-name openvpn-personal \
    --profile $AWS_VPN_PROFILE \
    --parameter-overrides SSHKeyName=$SSHKEYPAIR \
    --capabilities  CAPABILITY_IAM
```

### [Optional] Access OpenVPN EC2 Instance

```bash
OPENVPNIP=`aws cloudformation describe-stacks \
  --stack-name openvpn-personal \
  --profile $AWS_VPN_PROFILE \
  --output text \
    | grep OpenVPNEIP \
    | sed s/^.*EIP// \
    | tr -d '[:blank:]'`
ssh -i $SSHKEYPAIR.pem ec2-user@$OPENVPNIP
```
