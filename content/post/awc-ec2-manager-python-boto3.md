---
date: '2025-04-11T18:13:00+01:00'
draft: false
title: "Never Pay for Forgotten AWS EC2 Instances Again: A Python Boto3 Script to Check and Terminate EC2 Instances"
tags: ['AWS, 'Billing', 'Cost Optimisation', 'EC2', 'Script', 'Python']
categories: ['AWS']
summary: "This article shows how to leverage a Python script that uses AWS SDK (Boto3) to check EC2 instance status across all regions. Learn to find, stop, and terminate forgotten instances to avoid unexpected AWS bills"
thumbail: "images/python-ec2-cost.png"
---
## Problem Statement
Are you one of those who ends up getting an unexpected AWS bill because you forgot to terminate or stop one or more EC2 instances? Have you ever launched test instances across multiple regions and then struggled to find them all? 

## The Solution: 
I have been there, so I've created a Python script that leverages AWS SDK (Boto3) to help manage your EC2 instances across all regions. This tool only:
- Scans all AWS regions for EC2 instances at once.
- Displays their status with color-coding (green for running, red for stopped).
- Allows bulk operations (terminate all, stop all).
- Supports targeted actions on specific instances (terminate and/or stop specific EC2 instances).

The best part? You can run this script from your local machine anytime to get a complete picture of your AWS EC2 footprint and prevent unexpected charges from forgotten instances.

## Prerequisites
Before using this script, you'll need:

1. Python Installation
Since I am using Fedora distribution on my machine, you can install Python with:
```bash
sudo dnf install python3 python3-pip
```

2. Required Python Packages
```bash
pip3 install boto3 colorama argparse
```

3. AWS CLI Installation
```bash
sudo dnf install awscli
```

4. AWS Credentials Configuration
```bash
aws configure
```
You'll need to enter your `AWS Access Key`, `Secret Key`, `default region`, and output format.

5. IAM Permissions

For this script to work properly, your AWS user needs sufficient permissions to list, stop, and terminate EC2 instances across all regions.

In my example, I'm using a user called "devops-user" with the `AdministratorAccess` policy, which provides full access to all AWS services and resources.

However, for production environments, you should follow the principle of least privilege. The minimum permissions needed are:

- DescribeInstances (to list all instances).
- DescribeRegions (to find all regions).
- StopInstances (to stop running instances).
- TerminateInstances (to terminate instances).

Please refer to the AWS IAM documentation for futher details. 

## Setting up the Script
```bash
# Create a directory for your scripts
mkdir -p ~/Documents/DevOps_Projects

# Create the script file
vim ~/Documents/DevOps_Projects/ec2_manager.py
```
Now paste the script code (see below), save, and make it executable:
```bash
chmod +x ~/Documents/DevOps_Projects/ec2_manager.py
```

## The EC2 Mannager Script
```python
import boto3
import argparse
import concurrent.futures
from colorama import Fore, Style, init

# Initialize colorama
init()

class CustomHelpFormatter(argparse.HelpFormatter):
    def _format_args(self, action, default_metavar):
        if action.nargs == '+':
            return '[INSTANCE_ID...]'
        return super()._format_args(action, default_metavar)

def get_all_regions():
    """Get list of all AWS regions."""
    ec2_client = boto3.client('ec2', region_name='us-east-1')
    regions = [region['RegionName'] for region in ec2_client.describe_regions()['Regions']]
    return regions

def get_instances_by_region(region):
    """Get all EC2 instances in a specific region."""
    ec2_client = boto3.client('ec2', region_name=region)
    response = ec2_client.describe_instances()
    instances = []
    
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            instances.append({
                'InstanceId': instance['InstanceId'],
                'State': instance['State']['Name'],
                'Region': region,
                'Type': instance['InstanceType'],
                'Name': next((tag['Value'] for tag in instance.get('Tags', []) if tag['Key'] == 'Name'), 'No Name')
            })
    
    return instances

def list_all_instances():
    """List all instances across all regions with improved performance."""
    regions = get_all_regions()
    all_instances = []
    
    print(f"Checking {len(regions)} AWS regions for EC2 instances...")
    
    # Use ThreadPoolExecutor to check regions in parallel
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        region_instances = list(executor.map(get_instances_by_region, regions))
    
    # Flatten the list of lists and display results
    for instances in region_instances:
        if instances:
            region = instances[0]['Region']
            print(f"\nInstances in {region}:")
            for instance in instances:
                state = instance['State']
                if state == 'running':
                    state_color = Fore.GREEN
                elif state == 'stopped':
                    state_color = Fore.RED
                else:
                    state_color = Fore.YELLOW
                
                print(f"Name: {instance['Name']} | Instance ID: {instance['InstanceId']} | Type: {instance['Type']} | State: {state_color}{state}{Style.RESET_ALL}")
            
            all_instances.extend(instances)
    
    print(f"\nTotal: {len(all_instances)} instances found.")
    return all_instances

def stop_instances(instance_ids):
    """Stop specified EC2 instances."""
    regions = get_all_regions()
    stopped_instances = []
    
    for region in regions:
        ec2_client = boto3.client('ec2', region_name=region)
        region_instances = get_instances_by_region(region)
        region_instance_ids = [i['InstanceId'] for i in region_instances if i['InstanceId'] in instance_ids]
        
        if region_instance_ids:
            try:
                response = ec2_client.stop_instances(InstanceIds=region_instance_ids)
                for instance in response['StoppingInstances']:
                    print(f"Stopping instance {instance['InstanceId']} in {region}")
                    stopped_instances.append(instance['InstanceId'])
            except Exception as e:
                print(f"Error stopping instances in {region}: {e}")
    
    return stopped_instances

def terminate_instances(instance_ids):
    """Terminate specified EC2 instances."""
    regions = get_all_regions()
    terminated_instances = []
    
    for region in regions:
        ec2_client = boto3.client('ec2', region_name=region)
        region_instances = get_instances_by_region(region)
        region_instance_ids = [i['InstanceId'] for i in region_instances if i['InstanceId'] in instance_ids]
        
        if region_instance_ids:
            try:
                response = ec2_client.terminate_instances(InstanceIds=region_instance_ids)
                for instance in response['TerminatingInstances']:
                    print(f"Terminating instance {instance['InstanceId']} in {region}")
                    terminated_instances.append(instance['InstanceId'])
            except Exception as e:
                print(f"Error terminating instances in {region}: {e}")
    
    return terminated_instances

def get_instance_ids_by_state(state, all_instances=None):
    """Get instance IDs for instances in a specific state."""
    if all_instances is None:
        all_instances = []
        regions = get_all_regions()
        for region in regions:
            all_instances.extend(get_instances_by_region(region))
    
    return [instance['InstanceId'] for instance in all_instances if instance['State'] == state]

def main():
    parser = argparse.ArgumentParser(
        description='AWS EC2 Instance Manager',
        formatter_class=CustomHelpFormatter
    )
    parser.add_argument('--list', action='store_true', help='List all EC2 instances')
    parser.add_argument('--terminate-all', action='store_true', help='Terminate all EC2 instances')
    parser.add_argument('--terminate-running', action='store_true', help='Terminate only running EC2 instances')
    parser.add_argument('--terminate-stopped', action='store_true', help='Terminate only stopped EC2 instances')
    parser.add_argument('--terminate-specific', nargs='+', 
                        help='Terminate specific EC2 instances (provide one or more instance IDs)')
    parser.add_argument('--stop-all', action='store_true', help='Stop all running EC2 instances')
    parser.add_argument('--stop-specific', nargs='+',
                        help='Stop specific EC2 instances (provide one or more instance IDs)')
    
    args = parser.parse_args()
    
    # List all instances by default if no action is specified
    if not any(vars(args).values()):
        args.list = True
    
    # Get all instances information
    all_instances = list_all_instances() if args.list or args.terminate_all or args.terminate_running or args.terminate_stopped or args.stop_all else []
    
    # Process terminate commands
    if args.terminate_all:
        instance_ids = [instance['InstanceId'] for instance in all_instances]
        if instance_ids:
            confirm = input(f"Are you sure you want to terminate ALL {len(instance_ids)} instances? (yes/no): ")
            if confirm.lower() == 'yes':
                terminate_instances(instance_ids)
            else:
                print("Termination canceled.")
    
    elif args.terminate_running:
        running_instances = get_instance_ids_by_state('running', all_instances)
        if running_instances:
            confirm = input(f"Are you sure you want to terminate all {len(running_instances)} RUNNING instances? (yes/no): ")
            if confirm.lower() == 'yes':
                terminate_instances(running_instances)
            else:
                print("Termination canceled.")
        else:
            print("No running instances found.")
    
    elif args.terminate_stopped:
        stopped_instances = get_instance_ids_by_state('stopped', all_instances)
        if stopped_instances:
            confirm = input(f"Are you sure you want to terminate all {len(stopped_instances)} STOPPED instances? (yes/no): ")
            if confirm.lower() == 'yes':
                terminate_instances(stopped_instances)
            else:
                print("Termination canceled.")
        else:
            print("No stopped instances found.")
    
    elif args.terminate_specific:
        instance_ids = args.terminate_specific
        confirm = input(f"Are you sure you want to terminate the following instances: {', '.join(instance_ids)}? (yes/no): ")
        if confirm.lower() == 'yes':
            terminate_instances(instance_ids)
        else:
            print("Termination canceled.")
    
    # Process stop commands
    elif args.stop_all:
        running_instances = get_instance_ids_by_state('running', all_instances)
        if running_instances:
            confirm = input(f"Are you sure you want to stop all {len(running_instances)} running instances? (yes/no): ")
            if confirm.lower() == 'yes':
                stop_instances(running_instances)
            else:
                print("Stop operation canceled.")
        else:
            print("No running instances found.")
    
    elif args.stop_specific:
        instance_ids = args.stop_specific
        confirm = input(f"Are you sure you want to stop the following instances: {', '.join(instance_ids)}? (yes/no): ")
        if confirm.lower() == 'yes':
            stop_instances(instance_ids)
        else:
            print("Stop operation canceled.")

if __name__ == "__main__":
    main()
```

## How to Use the Script
### Viewing Available Options
First, let's see what options are available:
```bash
python3 ec2_manager.py --help
```
This displays all available commands:
```bash
usage: ec2_manager.py [-h] [--list] [--terminate-all] [--terminate-running] [--terminate-stopped] [--terminate-specific [INSTANCE_ID...]] [--stop-all] [--stop-specific [INSTANCE_ID...]]
AWS EC2 Instance Manager
options:
  -h, --help            show this help message and exit
  --list                List all EC2 instances
  --terminate-all       Terminate all EC2 instances
  --terminate-running   Terminate only running EC2 instances
  --terminate-stopped   Terminate only stopped EC2 instances
  --terminate-specific [INSTANCE_ID...]
                        Terminate specific EC2 instances (provide one or more instance IDs)
  --stop-all            Stop all running EC2 instances
  --stop-specific [INSTANCE_ID...]
                        Stop specific EC2 instances (provide one or more instance IDs)
```

### Listing All Instances
To see all your EC2 instances across all regions:
```bash
python3 ec2_manager.py --list
```

This produces output like:

```bash
Checking 17 AWS regions for EC2 instances...

Instances in eu-west-2:
Name: vm-test-03 | Instance ID: i-037bf9b351507311 | Type: t2.micro | State: running
Name: vm-test-04 | Instance ID: i-06c4b0620ef1f4510 | Type: t2.micro | State: running

Instances in sa-east-1:
Name: vm-test-01 | Instance ID: i-031f5aa905232b0bb | Type: t2.micro | State: running
Name: vm-test-02 | Instance ID: i-03572964b8ba53d4 | Type: t2.micro | State: stopped
Name: vm-test-03 | Instance ID: i-01622116f476dc5a5 | Type: t2.micro | State: running

Total: 5 instances found.
```

### Terminating a Specific Instance
Now let's see how to terminate a specific instance:
```bash
python3 ec2_manager.py --terminate-specific i-031f5aa905232b0bb
```
This produces the following output:

```bash
Are you sure you want to terminate the following instances: i-031f5aa905232b0bb? (yes/no): yes
Terminating instance i-031f5aa905232b0bb in sa-east-1
```

This command first displays all instances across all regions, then asks for confirmation before proceeding with termination
To verify the instance's new status after termination, we can run the `--list` command again to see the updated state.

### Terminating Only Running Instances
To terminate only instances that are currently running:
```bash
python3 ec2_manager.py --terminate-running
```
This command first displays all instances across all regions, then asks for confirmation before proceeding with termination
To verify the instance's new status after termination, we can run the `--list` command again to see the updated state.

### Terminating All Instances
To terminate all instances across all regions:
```bash
python3 ec2_manager.py --terminate-all
```
This command first displays all instances across all regions, then asks for confirmation before proceeding with termination
To verify the instance's new status after termination, we can run the `--list` command again to see the updated state.

### Stopping All Running Instances
To stop all instances that are currently running:
```bash
python3 ec2_manager.py --stop-all
```
This command first displays all instances across all regions, then asks for confirmation before proceeding with termination
To verify the instance's new status after termination, we can run the `--list` command again to see the updated state.

### Stopping Specific Instances
```bash
python3 ec2_manager.py --stop-specific i-037bf9b351507311 i-06c4b0620ef1f4510
```

This command directly asks for confirmation before stopping the specified instances
To verify the instances' new status after stopping, we can run the --list command again to see the updated state.
