# Steps-to-setup-EC2-instance-on-AWS
Steps to setup an EC2 instance on AWS with an Nginx proxy server in front. 

This is a brief guide to be used as reference. More detailed info about creating an instance is provided by Amazon at https://docs.aws.amazon.com/ec2/

## Intial Setup

1. Create and launch your EC2 instance on AWS (Need AWS account - free 1 year trial)
   - Swift servers work well with Ubuntu 18.04

1. Create a security group if you do not have one.

1. Open up port 80 for HTTP requests to `0.0.0.0/0` and `::/0`

1. Open up port 443 for HTTPS requests to `0.0.0.0/0` and `::/0`

1. Open up your server port (8080 is typlical for an api server) to `0.0.0.0/0`

1. Attach your security group to your instance(s)

1. Optional - add database if needed

1. create a ssh key 
   - I like to create a ssh host shortcut in the `~/.ssh/config`

1. Mark your ssh private key as read-only otherwise AWS will refuse the connection
   - `chmod 600 ~/.ssh/key` or `chmod 600 /path/to/your/ssh/key` 
   
1. ssh into instance and update system
   - `apt-get update`
   - `apt-get upgrade`
