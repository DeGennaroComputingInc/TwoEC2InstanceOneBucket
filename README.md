# Purpose
This README.md file is meant to record my thoughts as well as document my design and how I accomplished it.  
## Table of content

- [Prologue](#prologue)
- [Plan](#plan)
- [Requirements](#requirements)
- [Deliverables](#deliverables)
- [Resources](#resources)
- [Future plans](#future)
- [Addendum](#addendum)

## Prologue 
Anywhere in this README, the CloudFormation template or any scripts you find the letters 'HR', it stands for 'Help Received' and documents that what follows is not my work product or at least heavily borrowed from the source cited.
Warning: This template was tested in an account and region with a default VPC, subnets, security groups, etc.  Your mileage may vary if this does not apply to you.
## Plan
The plan is to create a CloudFormation template that contained all the infrastructure and would only require two parameters, the availability zone the EC2 instances will launch in and the name of the S3 bucket.  
### EC2 Instances
Initially, the design would be to have a parameter (EC2InstanceAZ) which would specify the availability zone the EBS volumes (InstanceASecondVolume, InstanceBSecondVolume, InstanceBThirdVolume) and EC2 instances (InstanceA, InstanceB) would be created in, create the volumes separate from their instances but add them to the "Volumes" key in the instances' "Properties" then perform any raid/filesystem/parition actions in User Data.  Finally, a cron job to either upload or download a test file (file.txt) to the S3 bucket will be created before rebooting the instances  
### S3 Buckets 
For the S3 bucket, I enabled SSE with aws:kms to have data encrypted at rest and created a policy with Resource-specific statements for the global AWS condition SecureAccess; the instance should be able to access the bucket since it is in the same account as long as the instance has an instance profile
## Instructions
### Using the AWS free tier in the N. Virginia region, automate all steps necessary to complete the following exercise:

#### In your personal Virtual Private Cloud, create two EC2 instances using the type t2.micro.
The CloudFormation template should be launched in us-east-1 in an account with a default VPC (classic accounts not supported) so that the default subnets, security groups, etc, are used; the AZ in the parameter specfied will be used to make sure the EC2 instance and EBS volumes are created in the same AZ.  Note: t2.micro are not nitro so the EBS volumes are not NVMe EBS volumes.   

#### Instance “A”:
##### Operating system image 0080e4c5bc078760e
Searching through the public AMIS, I found the one below with the following details:
```
{
    "Images": [
        {
            "Architecture": "x86_64",
            "CreationDate": "2018-11-28T21:08:11.000Z",
            "ImageId": "ami-0080e4c5bc078760e",
            "ImageLocation": "amazon/amzn-ami-hvm-2018.03.0.20181129-x86_64-gp2",
            "ImageType": "machine",
            "Public": true,
            "OwnerId": "137112412989",
            "State": "available",
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/xvda",
                    "Ebs": {
                        "DeleteOnTermination": true,
                        "SnapshotId": "snap-01d81204beb02804b",
                        "VolumeSize": 8,
                        "VolumeType": "gp2",
                        "Encrypted": false
                    }
                }
            ],
            "Description": "Amazon Linux AMI 2018.03.0.20181129 x86_64 HVM gp2",
            "EnaSupport": true,
            "Hypervisor": "xen",
            "ImageOwnerAlias": "amazon",
            "Name": "amzn-ami-hvm-2018.03.0.20181129-x86_64-gp2",
            "RootDeviceName": "/dev/xvda",
            "RootDeviceType": "ebs",
            "SriovNetSupport": "simple",
            "VirtualizationType": "hvm"
        }
    ]
}
```
Tihs means the root volume will be remapped using `/dev/xvda`.
##### The root volume should be 9 GB in size.
By adding a `/dev/xvda` to the BlockDeviceMappings with the Size set to 9, this created a root volume of 9GB as the AMI is an EBS one, not an instance store.
##### Create an additional EBS volume
This was accomplished by adding an AWS::EC2::Volume to the Resources's values and having InstanceA DependsOn it; InstanceA has a Volumes property that Refs it. Because it was the second EBS volume, the Device value was set to `/dev/sdf`. [HR](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html)
###### No encryption.
By default, EBS volumes are not encrypted so not specifying the Encrypted key was sufficient.
###### Volume type gp2 with 1 GB of storage.
By default, not specifying VolumeType means the EBS volume will be gp2; alternatively, the Size was set to 1
###### File system of XFS.
In user data, we install the tools necessary to make an XFS filesystem, find the device by the suffix specified in path and then made the filesystem. [HR](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-initialize.html)
##### Create the directory /dataup
We created the directory with mkdir -p in case it already exists (should probably fail though if it does).
##### Mount the volume.
Adding the volume to /etc/fstab and then executing mount -a makes sure the volume is mounted after reboot
##### Reboot the instance.
Finally, we reboot the instance using the AWS CLI and the metadata service; the command should use the permissions from the instance profile attached to the EC2 instance.

#### Instance B
##### Operating system image 05a36d3b9aa4a17ac
Below is the AMI for instance B
```
{
    "Images": [
        {
            "Architecture": "x86_64",
            "CreationDate": "2018-10-23T12:22:10.000Z",
            "ImageId": "ami-05a36d3b9aa4a17ac",
            "ImageLocation": "099720109477/ubuntu/images/hvm-ssd/ubuntu-trusty-14.04-amd64-server-20181022",
            "ImageType": "machine",
            "Public": true,
            "OwnerId": "099720109477",
            "State": "available",
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/sda1",
                    "Ebs": {
                        "DeleteOnTermination": true,
                        "SnapshotId": "snap-0531866303aed18cc",
                        "VolumeSize": 8,
                        "VolumeType": "gp2",
                        "Encrypted": false
                    }
                },
                {
                    "DeviceName": "/dev/sdb",
                    "VirtualName": "ephemeral0"
                },
                {
                    "DeviceName": "/dev/sdc",
                    "VirtualName": "ephemeral1"
                }
            ],
            "Description": "Canonical, Ubuntu, 14.04 LTS, amd64 trusty image build on 2018-10-22",
            "Enasupport": true,
            "Hypervisor": "xen",
            "Name": "ubuntu/images/hvm-ssd/ubuntu-trusty-14.04-amd64-server-20181022",
            "RootDeviceName": "/dev/sda1",
            "RootDeviceType": "ebs",
            "SriovNetSupport": "simple",
            "VirtualizationType": "hvm"
        }
    ]
}
```
Similarly as Instance A, the root volume will be remapped using `/dev/sda1`.
##### The root volume should be 9 GB in size.
Again, by adding the root path to the BlockDeviceMappings, the root volume was 9GB.
##### Create two EBS volume and configure for RAID 0
Similar as Instance A but used `/dev/sdg` in addition to `/dev/sdf`. 
###### Encrypted.
One difference between the additional EBS Volumes from Instance A to Instance B is Encrypted is set to true so the EBS volumes are encrypted by the default KMS key for EBS.
###### Volume type gp2 with 1 GB of storage.
Same as Instance A.
###### File system of XFS.
First raided the two EBS volumes in RAID 0 [HR](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/raid-config.html); this was particularly problematic because the software repositories for this AMI contained corrupted packages [HR](https://askubuntu.com/questions/248970/how-to-install-dracut-on-ubuntu) which couldn't be remediated by adding newer software repositories as it only required more and more dependencies not available in the older repositories.  The solution was to forcefully remove one 'corrupt' package, manually download the deb for the package needed and then install it using dpkg, not aptitude/apt-get.  However, once this was done, dracut would not seem to make a proper initramfs image (or the Ubuntu instance would not reboot for some other reason after reboot).  Per [HR](https://vsupalov.com/aws-ebs-raid0/), update-initramfs is another way of creating this initramfs image; this allowed the instance to reboot properly. 
##### Create the directory /datadown
Similar to instance A.
##### Mount the volume.
This was accomplished similarly to instance A as some options that allow for booting the instance regardless of the array state were added.
##### Reboot the instance.
Similarly to Instance A, use the metadata service and an instance profile; however, this did require installing the AWS CLI.

#### Create an S3 bucket
Warning: This bucket will not be accessible from the console as the s3:ListAllMyBuckets permission is not included 
##### The bucket policy should enforce data encryption in flight.
I added two statements to the bucket policy; one that denies all actions at the bucket level and another that denies all actions at the object level.  Each of these statements have an exception: if the communication is over HTTPS, the action may proceed.  As the ACL of the bucket is for the creator (ie the account the CloudFormation template is launched in) and assuming that sane ACLs are set for the objects within the bucket, principals that have sufficient permissions other than the bucket policy should be able to access the bucket but only over HTTPS. [HR](https://aws.amazon.com/blogs/security/how-to-use-bucket-policies-and-apply-defense-in-depth-to-help-secure-your-amazon-s3-data/)
##### The bucket should have encryption for objects at rest.
In the Bucket's resources, I set the BucketEncryption so that all objects put to the bucket are encrypted using the KMS CMK for S3. [HR](https://docs.aws.amazon.com/AmazonS3/latest/dev/bucket-encryption.html)
 
#### Using the AWS CLI:
##### On instance “A” create a test file in the directory /dataup and upload it to the S3 bucket. 
Using here-to docs and cron, the instance A puts a file in /dataup to the bucket every minute.
##### On instance “B” pull the test file from the S3 bucket to the /datadown directory.
Consequently, instance B does the same except getting the file to /datadown every minute.
## Deliverables
- TwoEC2InstancesOneBucket.json
## Resources 
### AWS
#### AWS Docs
- [AWS::EC2::Instance](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html)
- [AWS::EC2::Instance BlockDeviceMappings](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-blockdev-mapping.html)
- [AWS::EC2::Volume](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-ebs-volume.html#cfn-ec2-ebs-volume-size)
- [AWS::S3::Bucket](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html)
- [AWS::S3::BucketPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-policy.html)
- [AWS::S3::Bucket BucketEncryption](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket-bucketencryption.html)
- [AWS::IAM::Role](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html)
- [Default VPC and default subnets (this template was tested in an account and region with default VPCs and subnets; YMMV if this does not apply to you)](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html#create-default-vpc)
- [Amazon EC2 Instance Store](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)
- [What is the difference between volume and blockdevicemappings in CloudFormation?](https://stackoverflow.com/questions/37372693/what-is-the-difference-between-volume-and-blockdevicemapping-tags-in-ec2-cloudfo)
- [Device Naming on Linux Instances (see the table for HVM)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html)
- [RAID Configuration on Linux (in particular the Creating a RAID Array on Linux section was helpful although mdadm conf path on Ubuntu is /etc/mdadm/mdadm.conf, fstab requires nofail and nobootwait for this version of Ubuntu and dracut was not useful)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/raid-config.html) 
- [What is the difference between 'nobootwait' and 'nofail' in fstab? (This notes the historical changes from Ubuntu 14 to 16)](https://unix.stackexchange.com/questions/53456/what-is-the-difference-between-nobootwait-and-nofail-in-fstab)
- [Initializing Amazon EBS Volumes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-initialize.html) 
- [CloudFormation Parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)
- [How to Use Bucket Policies and Apply Defense-in-Depth to Help Secure Your Amazon S3 Data (Useful for the encrypted at rest (using AWS KMS keys to encrypt the objects) and during transit (by enforcing HTTPS in the bucket policy)](https://docs.aws.amazon.com/AmazonS3/latest/dev/using-with-s3-actions.html#using-with-s3-actions-related-to-bucket-subresources)
- [S3 Permission Policy(Useful when writing a bucket policy ie what S3 Actions correspond to which S3 Resource at the bucket,sub-bucket/object level)](https://docs.aws.amazon.com/AmazonS3/latest/dev/using-with-s3-actions.html#using-with-s3-actions-related-to-bucket-subresources)
### Ubuntu
- [Creating a Level 0 RAID from EBS Volumes on an AWS EC2 Instance (where the update-initramfs command comes from)](https://vsupalov.com/aws-ebs-raid0/)
- [Managing the initial ramfs (initramfs) archive (compares initramfs and dracut, various useful commands for exploring initramfs contents, etc)](https://kernel-team.pages.debian.net/kernel-handbook/ch-initramfs.html)
- [How to install dracut on Ubunutu?](https://askubuntu.com/questions/248970/how-to-install-dracut-on-ubuntu)
#### Cron
- [How to create a cron job using Bash automatically without the interactive editor? (gave me the idea to use hereto docs to echo into a file in /etc/cron.d/)](https://stackoverflow.com/a/42774010)
- [How to get CRON to call in the correct PATHs (useful when aws cli binary wasn't found on Ubuntu instance, need to provide PATH in cronjob)](https://stackoverflow.com/questions/2388087/how-to-get-cron-to-call-in-the-correct-paths)
## Future plans
TODO Make userdata and parameters more robust for accepting multiple EBS volumes that may be different named/different block devices and may have non-valid filesystems per file -s.
## Addendum
### CLI command
#### Launch the CloudFormation template
```
#This assumes you have your credentials in the [default] profile in ~/.aws/config or ~/.aws/credentials 
aws cloudformation create-stack --stack-name twoinstancesones3bucket --template-body file:///Path/to/the/template/TwoEC2InstancesOneBucket.json --parameters ParameterKey=AZ,ParameterValue=your-az-1a -ParameterKey=S3BucketName,ParameterValue=uniques3bucketname --capabilities "CAPABILITY_NAMED_IAM"
```
