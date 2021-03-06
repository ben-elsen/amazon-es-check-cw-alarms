
Amazon Elasticsearch Service Stats and Alarms
====================
This is a set of scripts that addresses some common needs found in a complex Amazon Elasticsearch Service environment. Currently it consists of 2 subcomponents:

1. es-create-cwalarms.py: script that creates a set of recommended CloudWatch alarms for specific Amazon ES metrics.
2. es-checkcwalarms-lambda.py: script that can be run stand-alone or as a scheduled Lambda to check whether the correct alarms are set. 

The scripts are provided as starting points; while they provide some generally useful functionality they likely will need customization to your specific environment and requirements.

The scripts are written in Python 2.7 and use Boto3.

Note that the term "domain" (the term used by Amazon Elasticsearch Service) and "cluster" (colloquially) are used interchangeably here.

es-check-cwalarms-lambda
------
This Python 2.7 script checks whether the recommended/desired alarms are set. It can be run from the command line; or, set up as a scheduled Lambda.  For example: if clusters were originally set up without the desired alarms; or, configuration changes - cluster size increases being a common one - might require updates to alarms.

The script assumes you have the permissions listed in the provided sample IAM role (es-create-cwalarms-iamrole.json).

The desired alarms are listed in an array embedded in the code (esAlarms). This ugly hack allows parameters to be updated based on current domain stats.
```
>python es-checkcwalarms.py -h
usage: es-checkcwalarms.py [-h] [-e ESPREFIX] [-n NOTIFY]
                                  [-f FREE] [-p PROFILE] [-r REGION]

Checks a set of recommended CloudWatch alarms for Amazon Elasticsearch Service domains (optionally, those beginning with a given prefix).

optional arguments:
  -h, --help            show this help message and exit
  -e ESPREFIX, --esprefix ESPREFIX
                        Only check Amazon Elasticsearch Service domains that begin with this prefix.
  -n NOTIFY, --notify NOTIFY
                        List of CloudWatch alarm actions; e.g. ['arn:aws:sns:xxxx']
  -f FREE, --free FREE  Minimum free storage (MB) on which to alarm
  -p PROFILE, --profile PROFILE
                        IAM profile name to use
  -r REGION, --region REGION
                        AWS region to check. Defaults to us-east-1
```
Sample of output: 
```
List of Amazon Elasticsearch Service domains starting with given prefix (awses): [u'awsestest53']
=======================================================================================================
Starting checks for Amazon Elasticsearch Service domain awsestest53 , version is 5.3
Automated snapshot hour is:  0
2 instances t2.small.elasticsearch
awsestest53 Does not have Dedicated Masters!!
awsestest53 Does not have Zone Awareness enabled!!
EBS enabled: True type: gp2 size (GB): 10 0 Iops; 20480  total storage (MB)
Desired free storage set to (in MB): 4096.0
awsestest53 Test-Elasticsearch-awsestest53-ClusterStatus.yellow-Alarm ClusterStatus.yellow Alarm ok; definition matches.
awsestest53 Test-Elasticsearch-awsestest53-CPUUtilization-Alarm Threshold does not match. Should be:  80.0 ; is 70.0
awsestest53 Test-Elasticsearch-awsestest53-CPUUtilization-Alarm AlarmActions does not match. Should be:  ['arn:aws:sns:us-west-2:123456789012:sendnotification'] ; is ['arn:aws:sns:us-west-2:123456789000:sendnotification']

```

To run as a Lambda:
Follow the instructions [here](https://docs.aws.amazon.com/lambda/latest/dg/get-started-create-function.html), with the following changes:
* For Runtime, use Python 2.7
* For IAM Role, use a Lambda execution role with the added permissions in the provided sample IAM role (es-create-cwalarms-iamrole.json)
* For Timeout: increase the value to 3 minutes (depending on the number of domains you have)
* If desired, add environment variables:
	* esprefix: only check domains beginning with this prefix
	* esfree: minimum free space 
* Edit the Lambda code inline, and paste in the contents of es-check-cwalarms.py.
 
 Save and test the Lambda. 
 
Es-create-cwalarms
-------
This Python 2.7 script can be run from the command line to create the recommended set of alarms for a specific Amazon ES cluster.

It assumes you have the permissions listed in the provided sample IAM role.

The alarms to be created are listed in an array embedded in the code (esAlarms). This ugly hack allows parameters to be updated based on current domain stats.
```
> python es-create-cwalarms.py -h
Starting ... at 2017-09-13 22:40:08.493000
usage: es-create-cwalarms.py [-h] -c CLUSTER -a ACCOUNT [-e ENV] [-n NOTIFY] [-f FREE] [-p PROFILE] [-r REGION]

Create a set of recommended CloudWatch alarms for a given Amazon Elasticsearch Service domain.

optional arguments:
  -h, --help            show this help message and exit
  -c CLUSTER, --cluster CLUSTER
                        Amazon Elasticsearch domain name (e.g., testcluster1)
  -a ACCOUNT, --account ACCOUNT
                        AWS account id of the owning account (needed for metric dimension).
  -e ENV, --env ENV     Environment (e.g., Test, or Prod). Prepended to the alarm name.
  -n NOTIFY, --notify NOTIFY
                        List of CloudWatch alarm actions; e.g. ['arn:aws:sns:xxxx']
  -f FREE, --free FREE  Minimum free storage (MB) on which to alarm
  -p PROFILE, --profile PROFILE
                        IAM profile name to use
  -r REGION, --region REGION
                        AWS region for the cluster.
> python es-create-cwalarms.py -c testcluster -a 123456789012 -e Test -p iamuser1 -r us-west-2
```
Check the results by going to the CloudWatch Alarms console, or by running es-checkcwalarms (see below). The naming convention for the alarms created is:  
```{Environment}-{domain}-{MetricName}-alarm```
