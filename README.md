# Description

A fictitious Retail customer loyalty application demo for single region and multi-region on AWS

This demo simulates a retail customer loyalty application workload of customers getting loyalty credits for their retail purchases.  The dbworkload Python simulation does this via 2 transactions, one to find the customer id for the given customer, and another to update the customer’s loyalty credits.

We start with a single region database simulation, using dbworkload, of performance for both customers located in the same region as the cluster and customers located in a different region from the cluster.  Performance for the local customers will, of course, be better than for the remote customers.

We then move onto a demo of a multi-regional cluster simulation using regional by rows tables and with dbworkload workloads running in each region to show that performance is the same for all users regardless of which region the user is located in.  This allows us to show off CockroachDB’s geo-locality functionality.  You can also scale up the load for performance demos using dbworkload’s parallelism capability.

The demo uses 2 tables, a customers table and a loyalty table, and each has 30 million rows, which is generated by dbworkload.

Also for the single region demo we will manually create the AWS environment to get you familiar with using AWS and for the multi-region demo we will use Ron Nollen’s multi-region Terraform github repository to create the multi-region environment.

# Files

|  File|Description  |
|--|--|
|haproxy.cfg  |Single region HAProxy config file  |
|customers_test.yaml  |yaml file to create test customers data file for 5 rows |
|customers.yaml  |yaml file to create customers data files for 30 million rows  |
|loyalty_test.yaml  |yaml file to create test loyalty data file for 5 rows   |
|loyalty.yaml  |yaml file to create loyalty data files for 30 million rows  |
|customers_test.csv  |5 rows of customers data to test import command |
|loyalty_test.csv  |5 rows of loyalty data to test import command |
|load_customers_test.sql  |SQL command to import customers test data |
|load_loyalty_test.sql  |SQL command to import loyalty test data |
|load_loyalty_aws.sql  |SQL commands to import all 30 million rows of loyalty data |
|load_customers_aws.sql  |SQL commands to import all 30 million rows of customers data |
|databae_build_script.sql  |SQL DDL commands to build database objects |
|build_2_tables_only.sql  |SQL DDL commands to built 2 tables only |
|add_loyalty_keys_to_customers_table.sql  |SQL to add customers UUID key to loyalty table  |
|ulta_beauty.py  |Python program to run using dbworkload |
|run_dbworkload.sh  |Bash script to kickoff dbworkload runs on the app node(s) |

# Hardware Configurations

## Single region demo:

I have used a two level architecture for this demo with the database cluster at one level, and the application server nodes as the second level with the application server nodes running both the proxy server (load balancer) and the dbworkload workload.

* Cluster nodes config: 3 EC2 nodes using m6a.2xlarge instances (8 cpu’s and 32 GB RAM with 120GB SSD)
	Each node will be in a single region and it’s own zone (you must pick a region with a minimum of 3 zones)
* Application server nodes config: 2 EC2 nodes using m6a.2xlarge instances (8 cpu’s and 32 GB RAM with 100GB SSD)
	One node will be in the same region as the cluster and the other will be in a much farther away region

My region choices were to create the cluster in us-east-2 (Ohio) and have the application server nodes in us-east-2 and us-west-2 (Oregon)

## Multi-region demo:

Ron’s Terraform scripts create a three level architecture.  One level for the database cluster, one level for the proxy server nodes, and a third level for the application server nodes, where you will run the dbworkload workload.  You must have all three levels.  (I have tried to create a 2 level config and it doesn’t work.). When you setup the Terraform script’s configuration file, make sure you pick AWS regions with a minimum of 3 zones; i.e. do NOT use us-west-1 (Northern California).

* Cluster nodes config: 9 EC2 nodes using m6a.2xlarge instances
	There will be 3 regions and 3 nodes per region with each node in its own region.
	Ron’s script will automatically create these for you in the right zones and regions.
* Proxy nodes config: 3 EC2 nodes using m6a.xlarge instances
	Each proxy node will be in a separate region.  Ron’s script will automatically create these for you in the right regions.
* Application server nodes config: 3 EC2 nodes using m6a.2xlarge instances
	Each application node will be in a separate region.  Ron’s script will automatically create these for you in the right regions.

My region choices were to use us-east-1 (North Virginia), us-east-2 (Ohio) and us-west-2 (Oregon).

# Setup

## General Setup

1. Create key pairs for each region you are using for your single region and mult-region clusters.  So you want to setup 1 key pair for each of the 3 regions you’ll be using.  I would suggest us-east-1 (North Virginia), us-east-2 (Ohio) and us-west-2 (Oregon)
2. Create a non-public S3 bucket where you will put your table data files that you generate with dbworkload.  You’ll later import these data files into your tables using the SQL IMPORT INTO command.  (Note: Fabio’s dbworkload github mentions doing this by using a web server on your application node.  With the latest versions of CRDB (I used version 24.2.2) this no longer will work if you have a large number of data files to import.)  Copy your bucket's URI; select the 'Copy S3 URI' button in the upper right corner of the S3 bucket screen.
3. Create an IAM policy to allow you to upload, download and access an S3 bucket.  After you press the 'create policy' button select the JSON tab on your upper right.  You'll want to use the JSON below as your template.  Only alter the Resource section.  You can see that I have 6 entries which are for 3 S3 buckets.  Remove my entries and create yours as follows: For each S3 URI create 2 entries; one that ends with a __/*__ and one that doesn't.  Make sure each entry starts with __arn:aws:__  And don't forget a comma at the end of each entry except for the last one.  Then click the 'Next' button in the lower right, enter your policy name, and create your policy.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::adam-london",
                "arn:aws:s3:::adam-london/*",
                "arn:aws:s3:::adam-london/demo/loyalty",
                "arn:aws:s3:::adam-london/demo/loyalty/*",
                "arn:aws:s3:::adam-london/demo/customer",
                "arn:aws:s3:::adam-london/demo/customer/*"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
```

4. Create an IAM user and access key.  Got to the IAM users screen and select the create user button.  When you get to the 'Set Permissions' screen, select the 'attach policies directly' choice and then select the policy you just created.  Then select the 'Next' button and create the user.

Now go back to the list of IAM users and click on your new user which is highlighted in blue.  Select the 'create access key' choice on the right.  On the "Access key best practices & alternatives" screen select the "Other" choice and click next.  On the next screen press the 'Create Access Key'.  Write down in a secure place your AWS access key and your AWS secret access key which you will need later.

## Single Region Setup

1. Create your Virtual Private Cloud.  Go to the VPC screen and make sure you have selected the AWS region that you want the VPC created in which is also where your database cluster will be located in.  Select the 'create VPC' button in the upper right.  Then do as follows:
   A. Select the' VPC and more' radio button.
   B. Under the name tag, clear the default entry of 'Project' and put in the name you want for your VPC and its related objects.  I used 'adam-london'.
   C. Record your CIDR block for later use.  You can leave the default of 10.0.0.0/16
   D. Change your 'Number of Availability Zones' to 3 and it will automatically update the number of private and public subnets.
   E. Take a screen snapshot of the 'Preview screen' on your right.  You'll need the names of your VPC and public and private subnets for later use.
   F. Create your VPC.

Here is a picture of my VPC preview screen:

![AWS VPC Sample Configuration](https://github.com/adamblondon01/dbworkload-demo/blob/859a7920f5c60df1a7c78d82dde2cf79217c4548/sample%20AWS%20VPC%20configuration.png)

3. Create your internal and external security groups.  Go to the 'Security Groups' screen and make sure you have selected the AWS region that you want these security groups created in, and correspondingly, your database cluster. (Look in the upper right corner for the region dropdown list.)  Then create the external security group.  Make sure you have inbound entries for ports 22, 8080 and 26257 for your home IP address and the Netskope IP addresses.  (See [Netskope IP Addresses](https://cockroachlabs.atlassian.net/wiki/spaces/HELP/pages/3735027747/Netskope+IP+Ranges) )  For your outbound entries, you can enable everything.





readme examples:

[test link](https://cockroachlabs.com)

![a sql file](build_2_tables_only.sql)
