# PS Regional Training 2021 AWS Labs


<img src="https://www.paloaltonetworks.com/content/dam/pan/en_US/images/logos/brand/primary-company-logo/Parent-logo.png" width=50% height=50%>

## Overview

This lab will involve deploying a solution for AWS using Palo Alto Networks VM-Series in the Gateway Load Balancer (GWLB) topology.

The lab assumes an existing Panorama that the VM-Series will bootstrap to. Panorama assumptions:
- Accessible with public IP on TCP 3978
- Prepped with Template Stacks and Device Groups
- vm-auth-key generated on Panorama

## Deployment

- Login to console
- Verify in correct region 

### Step x: Update IAM Policies


- Search for `IAM` in top searchbar (IAM is global)
- In IAM dashboard select Users -> awsstudent
- Expand `default_policy`, Edit Policy -> Visual Editor
- Find the Deny Action for `Cloud Shell` and click `Remove` on the right
- Review policy / save changes

```
"cloudshell:*"
```

Note: If using the json editor, you will syntax error if you leave trailing comma on line 32 after removing `cloudshell` 


<img src="https://user-images.githubusercontent.com/43679669/108144448-aa08ad00-7097-11eb-926d-66ab34e050da.png" width=50% height=50%>

### Step x: Launch CloudShell

- Search for `cloudshell` in top search bar

This lab will use cloudshell for access to AWS CLI and as a runtime environment to provision your lab resources in AWS using terraform. Cloudshell will have the same IAM role as your authenticated user and has some utilities (git, aws cli, etc) pre-installed. It is only available in limited regions currently.


- Check which Marketplace VM-Series images (AMIs) are available

This terraform deployment will look up the AMI ID to use for the deployment based on the variable `fw_version`. New AMIs are not always published for each minor release. Therefore, it is a good idea to verify what version AMI most closely matches your target version.

In cloud console, enter:

`aws ec2 describe-images --filters "Name=owner-alias,Values=aws-marketplace" --filters Name=name,Values=PA-VM-AWS-10* Name=product-code,Values=6njl1pau431dv1qxipg63mvah --region us-west-2`

How many different BYOL AMIs are avilable for 10.x in this region?

product-code is a global value that correlates with Palo Alto Networks marketplace offerings. This is global and the same across all regions. There will be changes to this as vm-flex offerings come live.

```
    "byol"  = "6njl1pau431dv1qxipg63mvah"
    "payg1" = "6kxdw3bbmdeda3o6i1ggqt4km"
    "payg2" = "806j2of0qy5osgjjixq9gqc6g"
```
The name tag of the image should be standard and can be used for the filter. For example `PA-VM-AWS-9.1*`, `PA-VM-AWS-9.1.3*`, `PA-VM-AWS-10*`. This is the same logic the terraform will use to lookup the AMI based on the `fw_version` variable.

**We see that 10.0.4 AMI is availble, so we will use that for the variable**


- Generate SSH Key in cloudshell

Any EC2 Instance must be associated with a SSH keypair, which is the default method of initial interactive login to instances. With successful bootstrapping, there should not be any need to connect to the VM-Series instances direclty with this key, but it is usually good to keep this key securely stored for any emergency backdoor access. For this lab, a keypair will be generated in the cloudshell and then terraform will create a corresponding object in AWS using the same key.

`ssh-keygen -f ~/.ssh/ps-lab -t rsa -C ps-lab`

- Download ssh key for use later to SSH to instances

Actions -> Download File -> Path: `/home/cloudshell-user/.ssh/ps-lab`

TODO: handle key differently, some will prefer PPK. Use QL default key that is created, we can look it up in terraform.


### Step x: Clone the Repository

- Download Terraform in Cloudshell

TODO: Replace with one-liner

```
wget https://releases.hashicorp.com/terraform/0.13.6/terraform_0.13.6_linux_amd64.zip
unzip terraform_0.13.6_linux_amd64.zip
rm terraform_0.13.6_linux_amd64.zip
mv terraform /home/cloudshell-user/bin/
```

Verify Terraform is installed
`terraform --version`



```
$ git clone https://github.com/PaloAltoNetworks/ps-regional-2021-aws-labs.git
```

### Step x: Update tfvars


- //TODO add notes about terraform general usage, handling sensitive values, etc

For simplicity, only the variable values that need to be modified are separated into a separate tfvars file.

- Change into terraform directory 

`cd ps-regional-2021-aws-labs/terraform/vmseries/`

- Use vim to modify `student.auto.tfvars`


- Update the specifics of your deployment
- Anything marked with `###` should be replaced with appropriate value

We will be using the newer feature for light bootstrapping that does not require S3 buckets. Essentially, all of the paramaters normally specific in init-cfg can now be passed directly to the instance via user-data.

```
firewalls = [
  {
    name    = "vmseries01"
    fw_tags = {}
    bootstrap_options = {
      mgmt-interface-swap = "enable"
      plugin-op-commands  = "aws-gwlb-inspect:enable"
      type                = "dhcp-client"
      hostname            = "lab###_vmseries01"
      panorama-server     = "###"
      panorama-server-2   = "###"
      tplname             = "TPL-STUDENT-STACK-###"
      dgname              = "DG-STUDENT-###"
      vm-auth-key         = "###"
      authcodes           = "###"
      #op-command-modes    = ""
    }
    interfaces = [
      { name = "vmseries01-data", index = "0" },
      { name = "vmseries01-mgmt", index = "1" },
    ]
  },
  {
    name    = "vmseries02"
    fw_tags = {}
    bootstrap_options = {
      mgmt-interface-swap = "enable"
      plugin-op-commands  = "aws-gwlb-inspect:enable"
      type                = "dhcp-client"
      hostname            = "lab#_vmseries02"
      panorama-server     = "###"
      panorama-server-2   = "###"
      tplname             = "###"
      dgname              = "###"
      vm-auth-key         = "###"
      authcodes           = "###"
      #op-command-modes    = ""
    }
    interfaces = [
      { name = "vmseries02-data", index = "0" },
      { name = "vmseries02-mgmt", index = "1" },
    ]
  }
]
```

### Step x: Apply Terraform

- Terraform init / apply

```
Plan: 168 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

It should take 5-10 minutes for terraform to finish deploying all resources.

When complete, you will see a list of outputs. Copy these off locally so you can reference them in later steps. You can also come back to this directory in CloudShell and run `terraform output`. 



### Step x: Things to do while waiting on launch

All resources are now created in AWS, but it will be around 10 minutes until VM-Series are fully initialized and bootstrapped.

In the meantime, lets go look at what you built!


- Inspect VM-Series user data

EC2 Dashboard -> Instances -> Select `vmseries01` -> Actions -> Instance settings -> Edit user data

> Verify the values matches what was provided in your Lab Details

> What are some tradeoffs of using user-data method for bootstrap vs S3 bucket?

> What needs to happen if you have a typo or missed a value for bootstrap when you deployed?

---

- Get VM-Series instance screenshot

This can be useful to get a view of the console during launch. It is not interactive and must be manually refershed, but you can at least see some output related to bootstrap process or to troubleshoot if the VM-Series isn't booting properly or is in maintenance mode.

EC2 Dashboard -> Instances -> Select `vmseries01` -> Actions -> Monitor and troubleshoot -> Get instance screenshot

---

- Check VM-Series instance details

> What is the instance type? Which BYOL model(s) would this instance type be appropriate for?

> How many interfaces are associated to the VM-Series? Which interface is the default ENI for the instance? Which interfaces have public IPs associated?

> Check the security group associated with the "data" interface. What is allowed inbound? What is the logic of this SG?

> What Instance Profile was the VM-Series launched with? What actions does it allow? What are some other use-cases where you need to allow additional IAM permissions for the instance profile?

---

- Check cloudwatch bootstrap logs

Search for `cloudwatch` in the top search bar
Logs -> Log groups -> PaloAltoNetworksFirewalls

Assuming enough time has passed since launch, verify that the bootstrap operations completed successfully.

It is normal for the VMs to lose connectivity to Panorama initially after first joining.

> What is required to enable these logs during boot process?

---
- Look at VPC & TGW route tables, endpoints, correlate to the topology diagram


---
- Look at Load Balancers

Health probes of GWLB




### Step 50: Finished

Congratulations!

You have now successfully ….


Manual Last Updated: 2021-02-16
Lab Last Tested: -

