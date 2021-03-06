# zabbix-cloudwatch

[![Gem Version](https://badge.fury.io/rb/zabbix-cloudwatch.png)](http://badge.fury.io/rb/zabbix-cloudwatch)
[![Code Climate](https://codeclimate.com/github/randywallace/zabbix-cloudwatch.png)](https://codeclimate.com/github/randywallace/zabbix-cloudwatch)
[![Dependency Status](https://gemnasium.com/randywallace/zabbix-cloudwatch.png)](https://gemnasium.com/randywallace/zabbix-cloudwatch)
[![Build Status](https://travis-ci.org/randywallace/zabbix-cloudwatch.png?branch=master)](https://travis-ci.org/chrisfu/zabbix-cloudwatch)
[![Coverage Status](https://coveralls.io/repos/randywallace/zabbix-cloudwatch/badge.png)](https://coveralls.io/r/randywallace/zabbix-cloudwatch)

An external script for getting cloudwatch metrics into Zabbix

```
Usage: zabbix-cloudwatch

  -h, --help              This Message
  -n, --namespace         Namespace (AWS/Autoscaling, AWS/EC2, etc...)
  -m, --metricname        Metric Name (GroupInServiceInstances,EstimatedCharges, etc...)
  -d, --dimensions        Dimensions array (f.e.: [{ :name => 'dim1', :value => 'val1'}, { :name => 'dim2', :value => 'val2'}])
  -t, --monitoring-type   detailed|basic|manual                     Default: basic
  --seconds-ago           Start collecting data some seconds ago
  -s, --statistic         Minimum|Maximum|Average|Sum|SampleCount   Default: Average
  --aws-access-key        AWS Access Key
  --aws-secret-key        AWS Secret Key
  --aws-region            AWS Region                                Default: us-east-1
```

## Getting it running

* Currently tested against ruby 1.8.7, 1.9.3, and 2.0.0
* for some of the gem dependencies, you will need the ruby development packages, gcc, libxml2, and libxslt

Modify these steps to taste (examples given running on the Amazon AMI 2013.03):
```bash
yum install ruby ruby-devel rubygems gcc libxml2-devel libxslt-devel
gem install bundler zabbix-cloudwatch
ln -s $(which zabbix-cloudwatch) /var/lib/zabbixsrv/externalscripts/zabbix-cloudwatch

#We've made patches, lets fix the gem we just installed
gem contents zabbix-cloudwatch
#replace the bin/zabbix-cloudwatch & lib/zabbix-cloudwatch.rb in the gem files listed above with the local copies
#I'm not a ruby dev, so I'm not sure how to install this gem from a source dir

```

## Examples

```bash
zabbix-cloudwatch -n AWS/EC2 \
                  -m CPUUtilization \
                  -d AutoScalingGroupName \
                  -v your-auto-scaling-group \
                  -t detailed \
                  -s Sum
```

or

```bash
zabbix-cloudwatch -n AWS/ELB \
                  -m HealthyHostCount \
                  -d LoadBalancerName \ #This is supposed to be a key for a key value filter, below you define the value
                  -v NGINX #This should be your ELB name
```

## Creating the IAM User

The following actions need to be allowed in IAM for this script to work with the keys you provide:

```
"cloudwatch:DescribeAlarms"
"cloudwatch:GetMetricStatistics"
```

## AWS Credentials

There are (3) ways to get your AWS Credentials into `zabbix-cloudwatch`.

**Note that *none* of these options are "safe", so make sure you are using a set of IAM Keys with extremely restricted 
permissions.**

### 1. Environment Variables (which is difficult with Zabbix):
```bash
export AWS_ACCESS_KEY_ID="YOUR ACCESS KEY" 
export AWS_SECRET_ACCESS_KEY="YOUR SECRET ACCESS KEY"
export AWS_REGION="YOUR AWS REGION"
```

### 2. Within the binary in the gem.  

If you intend to do it this way, I suggest you make a copy of the binary
and place it in your zabbix externalscript path (instead of the suggested symlink in the installation example).

Find the binary like this:

```bash
ls $(gem env gemdir)/gems/zabbix-cloudwatch-$(zabbix-cloudwatch --version)/bin/zabbix-cloudwatch
```

And place it in your externalscripts path like this (your zabbix path/user/group may be different):

```bash
cp $(gem env gemdir)/gems/zabbix-cloudwatch-$(zabbix-cloudwatch --version)/bin/zabbix-cloudwatch \
      /var/lib/zabbixsrv/externalscripts/
chown zabbix:zabbix /var/lib/zabbixsrv/externalscripts/zabbix-cloudwatch
```

The class variables for this are at the very top of the file for your convenience.

### 3. Passing in your AWS Keys when you run zabbix-cloudwatch using the command line flags.

```bash
zabbix-cloudwatch -n AWS/AutoScaling \
                  -m GroupInServiceInstances \
                  -d AutoScalingGroupName \
                  -v your-auto-scaling-group \
                  --aws-access-key 'YOUR ACCESS KEY' \
                  --aws-secret-key 'YOUR SECRET KEY' \
                  --aws-region 'YOUR AWS REGION'
```

### 4. Get billing metrics for 5 hours

```bash
zabbix-cloudwatch -n AWS/Billing \
                  -m EstimatedCharges \
                  --dimensions "[{:name =>'ServiceName',:value =>'AmazonEC2'},{:name =>'LinkedAccount',:value =>'0123456789'},{:name =>'Currency',:value =>'USD'}]" \
                  --aws-access-key 'YOUR ACCESS KEY' \
                  --aws-secret-key 'YOUR SECRET KEY' \
                  --aws-region 'YOUR AWS REGION' \
                  --monitoring-type 'manual' \
                  --seconds-ago=18000
```

## Order of preference

The order of preference that this gem uses for the region and keys (individually) are:

* Commandline flag
* Within the binary
* Environment Variable

