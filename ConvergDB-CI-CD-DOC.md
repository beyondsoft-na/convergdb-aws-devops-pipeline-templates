#Convergdb CI/CD On AWS Instruction

This document will be the instruction of how to deploy convergdb in different envrionment with CI/CD on AWS codebuild and codepipeline.

##1. Preparation 

####1.1) Deployment and schema files

Refer to ConvergDB offical site (https://github.com/beyondsoft-na/convergdb/wiki) to define/extract your schema file and deployment file. In this instruction, we use books.schema and books.deployment file as example. 

* books.deployment

```json
s3_source "${env.CONVERGDB_ENV_NAME}" {
  relations {
    relation {
      dsd = "ecommerce.inventory.books_source"
      storage_format = "json"
      storage_bucket = "demo-source-us-west-2.beyondsoft.us"
    }
  }
}

athena "${env.CONVERGDB_ENV_NAME}"  {
  etl_job_name = "demo2"
  etl_job_schedule = "cron(0 7 * * ? *)"
  relations {
    relation {
      storage_format = "parquet"
      dsd = "ecommerce.inventory.books"
      inventory_source = "default"
      spark_partition_count = 2
    }
  }
}
```

* books.schema

```json
domain "ecommerce" {
  schema "inventory" {
    relation "books_source" {
      relation_type = base
      attributes {
        attribute "item_number" { data_type = integer }
        attribute "title"       { data_type = varchar(100) }
        attribute "author"      { data_type = varchar(100) }
        attribute "price"       { data_type = numeric(10,2) }
        attribute "stock"       { data_type = integer }
      }
    }

    relation "books" {
      relation_type = derived { source = "books_source" }
      partitions = ["part_id"]
      attributes {
        attribute "item_number" { data_type = integer }
        attribute "title"       { data_type = varchar(100) }
        attribute "author"      { data_type = varchar(100) }
        attribute "price"       { data_type = numeric(10,2) }
        
        attribute "part_id" {
          data_type = varchar(100)
          expression = "substring(md5(title),1,1)"
        }
        attribute "retail_markup" {
          data_type = numeric(10,2)
          expression = "price * 0.26"
        }
        attribute "source_file" { 
          data_type = varchar(100) 
          expression = "convergdb_source_file_name"
        }
      }
    }
  }
}
```

####1.2) Download convergdb.jar

Download convergdb.jar from https://github.com/beyondsoft-na/convergdb/blob/master/convergdb.jar

####1.3) Run convergdb locally

Create 3 directories indicate 3 stages and copy the 3 files (schema, deployment and jar) to the directories respectively, the tree struction should look like below:

```
- convergdb-dev-east-2/
	- books.deployment
	- books.schema
	- convergdb.jar
- convergdb-qa-west-1/
	- books.deployment
	- books.schema
	- convergdb.jar
- convergdb-prod-west-2/
	- books.deployment
	- books.schema
	- convergdb.jar
``` 
Make sure to separate the different region or account for different stages. In this case, we put region in the directories' names.


Create a shell script to help run generate convergdb for different stages, an example is showing below:

```bash
export AWS_PROFILE="default" #optional, set only if you need run with different credentials 
export CONVERGDB_ENV_NAME="dev" # set dev, qa, and prod for each satge respectively. 
java -jar convergdb.jar generate
```
save this shell script in each directory, make sure the `CONVERGDB_ENV_NAME` varaible is associated with each proper stage directory. For instance, we name the file below as **convergdb_run.sh** in directory **convergdb-qa-west-1/**

```bash
export CONVERGDB_ENV_NAME="qa" # set dev, qa, and prod for each satge respectively. 
java -jar convergdb.jar generate
```

Run this shell script in each directory to generate bootstrap, doc and terraform folders. 

####1.4) Deploy bootstrap (optional if backend settings already exsit)


After running `convergdb generate` (or equivalent), in each directory, run bootstrap, make sure the enter the correct region for each stage accordingly.

```bash
$ cd bootstrap
$ terraform init
...
$ terraform plan -out tf.plan
var.region
  Enter a value: us-west-2
...
$ terraform apply tf.plan
...
```

After step 1.4, two files created in terraform/ directory: **terrafrom.tf.json** and **variables.tf**


##2. Init CI/CD

####2.1) Clone Template
Clone the repo https://github.com/beyondsoft-na/convergdb-aws-devops-pipeline-templates and replace your deployment and schema files as needed. Use the new content to make a new repo in your github account. 

####2.2) Set tfvars accordingly
Find the variables.tf for each stages that were created in step 1.4, replace the values in dev.tfvars, qa.tfvarts and prod.tfvars respectively. 

##3. Codebuild for Dev

####3.1) Create Codebuild Project 
* Create a codebuild project in AWS, connect to your GitHub using OAuth. 
* Select the github repo in your Github account, you will see an option **Primary source webhook events** appare under **source** setting. 
* Mark **Rebuild every time a code change is pushed to this repository** , and set **Event Type** to `PULL_REQUEST_CREATED` ( add other event type as needed)
* This will allow the codebuild be triggered every time a PR created in the Github repo.
* Set **buildspec_pr.yml** as the buildspec.yml file. 
* set artifacts as needed (optional)

####3.2) Give permission to Codebuild role
Find the role created with Codebuild project and attach permission to it, we attached `admministratoraccess` policy to that role in this example

##4. Codepipeline for qa and prod

4.1) Create a codepipeline

* Create a codepipeline
* Connect your github account
* Select the repo
* Set Branch as `master`
* In Build setting, set Build Provider as **AWS CodeBuild** 
* Click `create project`, the console will direct you to a new page to create a new codebuild project
* Refer to step 3.1 to create a new codebuild beside:
	* DO NOT set webhook for this codebuild
	* Set **buildspec_merge_qa.yml** as the buildspec.yml file
* After codebuild, the console will direct you back to Deployment Stage of codepipeline setting
* Follow the Build stage setup above for Deployment stage, create a codebuild from the piepline setup console and use **buildspec_merge_prod.yml** as buildspec.yml file

4.2) Create a manual approve action before deployment stage

* Edit pipeline
* Edit deployment stage
* Add action group before deployment stage
* select `manual approval` in action provider
* select SNS arn (create one if needed)

##Summary
* A PR will trigger `codebuild_pr` in codebuild
* A master branch merge will trigger codepipeline for qa and prod stages
* A manual approval is required to proceed with prod stage 