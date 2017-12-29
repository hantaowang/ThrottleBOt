Instructions on how to use ThrottleBot:

1. To run Throttlebot, please create a configuration file. test_config.cfg serves as an example of how such a file would look. All parameters specified are mandatory and parameters that require multiple values should be comma-separated,without spaces

The "Basic" Section describes general Throttlebot parameters.

baseline_trials: The number of experiment trials to run for the baseline experiment
trials: The number of trials to run each of the stressed experiments (you might feel inclined to choose fewer trials than experiment_trials for faster experimenting
increments: Describes how many levels of stressing. An increment of 20 suggests you would stress up to 20% of the resource's maximum capacity.
stress_these_resources: The resources that are good to consider. Current options so far are only DISK, NET, CPU-CORES, and CPU-QUOTA. 
stress_these_services: The names of the services you would want Throttlebot to stress. To stress all services,simply indicate *. Throttlebot will blacklist any non-application related services by default
redis_host: The host where the Redis is located (Throttlebot uses Redis as it's data store)
stress_policy: The policy that is being used by Throttlebot to decide which containers to consider on each iteration.

The "Workload" section describes several Workload specific parameters. Throttlebot will run the experiment in this manner on each iteration.

type: Each implemented workload will have a type. Set the experiment name here. This might be deprecated later.
request_generator: An instance that generates requests to the application under test. There might be multiple of these instances. 
frontend: The host name (or IP address) where the application frontend is
additional_args: The names of any additional arguments that would be used by this workload
additional_arg_values: The values of the additional arguments (see additional_args above), listed in the same order as the argument names in additional_args
tbot_metric: The experiment could return several metrics, but this tells Throttlebot which metric to prioritize MIMRs by. There can only be a single metric here. Ensure that the metric is spelled identically as in your workload.py
performance_target: A termination point for Throttlebot. This is for Throttlebot to know when to stop running the experiments. This is not yet implemented.

Once the configuration is set, ensure Redis is up and running, and then start Throttlebot with the following command.

$ python run_throttlebot.py <config_file_name>

2. If necessary, set the "password" variables for your SSH keys. Without this, Throttlebot cannot execute commands on the virtual machines. They are located within remote_execution.py and measure_performance_MEAN_py3.py.


## Deploying a Kubernetes cluster on AWS
These instructions will cover launching a Kubernetes cluster  on AWS and deploying a simple application on this cluster.

### Tools
This guide recommends the use of [Kubernetes Operations](https://github.com/kubernetes/kops) (kops) to deploy and maintain the cluster.

Alternatives:
- [kube-aws](https://github.com/kubernetes-incubator/kube-aws) - requires external DNS name
- kube-up - deprecated shell script for launching clusters, not compatible with kubernetes 1.6+

source: https://kubernetes.io/docs/getting-started-guides/aws/

### Prerequisites
- kubectl - Kubernetes command-line tool
- kops
- aws-cli

On macOS with Homebrew:
```
$ brew install kubectl
$ brew install kops
$ brew install awscli
```

### Configure AWS User Permissions
Using kops requires the correct API credentials for your AWS account (AmazonEC2FullAccess, AmazonRoute53FullAccess, AmazonS3FullAccess, IAMFullAccess, AmazonVPCFullAccess).

```
$ aws iam create-group --group-name kops

$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

$ aws iam create-user --user-name kops
$ aws iam add-user-to-group --user-name kops --group-name kops
$ aws iam create-access-key --user-name kops
```
```
# set up the aws client to use the new IAM user
# in most cases, set region to 'us-west-1' and output forma to 'json'
$ aws configure

# Export the variables for kops use (make sure to add to ~/.bash_profile)
$ export AWS_ACCESS_KEY_ID=<access key>
$ export AWS_SECRET_ACCESS_KEY=<secret key>
```
source: https://github.com/kubernetes/kops/blob/master/docs/aws.md

### Set up cluster state storage
Create a S3 bucket to store all information regarding cluster configuration and state. For most instances, set `aws_region` to whichever region you choose in aws configure.
```
$ aws s3api create-bucket --bucket {bucket_name} --region {aws_region} --create-bucket-configuration LocationConstraint={aws_region}
$ export KOPS_STATE_STORE=s3://{bucket_name}
# (make sure to add to ~/.bash_profile)
```

### Launching a Kubernetes gossip-based cluster
Use the following command to launch your cluster. More info here: https://github.com/kubernetes/kops/blob/master/docs/cli/kops_create_cluster.md
```
# aws_zone is usually 'us-west-1a', 'us-west-1b', or 'us-west-1c'
$ kops create cluster {cluster_name}.k8s.local --zones {aws_zone} --yes
```
After a couple of minutes, you should be able to validate the cluster has been created.
```
$ kops validate cluster
```
You can delete your cluster using a simple command.
```
$ kops delete cluster {cluster_name}.k8s.local --yes
````
source: http://blog.arungupta.me/gossip-kubernetes-aws-kops/

### Deploying a simple multi-tier application on your cluster
The Kubernetes documentation covers how to launch a basic web application using PHP and Redis. The documentation also explains how to externalize an application.
https://kubernetes.io/docs/tutorials/stateless-application/guestbook/

Alternatively, one can pull the following repository and deploy the application with a single comand.
https://github.com/kubernetes/examples/tree/master/guestbook

First check that your cluster is configured properly.
```
$ kubectl cluster-info
```
The application can then be deployed with a single command.
```
$ kubectl create -f guestbook/all-in-one/guestbook-all-in-one.yaml
```
You can check your currently running services.
```
$ kubectl get services
```
You can also delete the application by running the following command.
```
$ kubectl delete -f guestbook/all-in-one/guestbook-all-in-one.yaml
```


## ThrottleBot Compatibility
The SSH public key for EC2 instances launched by kops defaults to ~/.ssh/id_rsa.pub.


### CPU
Did not run into major issues.

### Disk
Must change how to reference cgroups and individual containers in change_container_blkio() function in modify_resources.py.

Kubernetes organizes containers using pods so the path has to be changed to:
```
$ /sys/fs/cgroup/blkio/kubepods/burstable/{pod#}/{container_ID}*/blkio.throttle.read_bps_device
````

### Network

Approach (does not work): Identify the veth of individual containers and use tc with HTB queuing discipling to throttle network bandwidth.

Identify the veth of individual containers by checking the iflink of containers from shell.
```
$ cat /sys/class/net/eth0/iflink
```
source: https://superuser.com/questions/1183454/finding-out-the-veth-interface-of-a-docker-container

Set bandwidth using tc.
```
$ sudo tc qdisc add dev {veth_id} handle 1: root htb default 11
$ sudo tc class add dev {veth_id} parent 1: classid 1:1 htb rate 1kbps
$ sudo tc class add dev {veth_id} parent 1:1 classid 1:11 htb rate 1kbps
``` 

Other approaches:
- Tried other queuing disciplines
- Create custom [Docker bridges](https://docs.docker.com/engine/userguide/networking/default_network/build-bridges/)? (tricky/have to define bridge before container deployment)
