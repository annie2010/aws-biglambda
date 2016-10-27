# BigLambda: A Serverless Framework to Run Ad Hoc MapReduce Jobs

BigLambda is a cost-effective way to run ad hoc MapReduce jobs. The price-per-query model and ease of use make it very suitable for data scientists and developers alike. 

## Architecture

![alt text] (https://s3.amazonaws.com/smallya-test/bl-git.png "BigLambda Architecture")


## Features

* Close to "zero" setup time
* Pay per execution model for every job
* Cheaper than other data processing solutions
* Enables data processing within a VPC

## Languages
* Python 2.7 (active development)
* Node.js

The Python version is under active development and feature enhancement.

### Benchmark

To compare BigLambda with other data processing frameworks, we ran a subset of the Amplab benchmark. The table below has the execution time for each workload in seconds: 

Dataset

s3n://big-data-benchmark/pavlo/[text|text-deflate|sequence|sequence-snappy]/[suffix].

S3 Suffix   Scale Factor    Rankings (rows) Rankings (bytes)    UserVisits (rows)   UserVisits (bytes)  Documents (bytes)
/5nodes/    5               90 Million      6.38 GB              775 Million         126.8 GB             136.9 GB

Queries:

* Scan query  (90M Rows, 6.36GB of data)
* SELECT pageURL, pageRank FROM rankings WHERE pageRank > X   ( X= {1000, 100, 10} )

    * 1a) SELECT pageURL, pageRank FROM rankings WHERE pageRank > 1000   
    * 1b) SELECT pageURL, pageRank FROM rankings WHERE pageRank > 100   


* Aggregation query on UserVisits ( 775M rows, ~127GB of data)
    * 2a) SELECT SUBSTR(sourceIP, 1, 8), SUM(adRevenue) FROM uservisits GROUP BY SUBSTR(sourceIP, 1, 8)


NOTE: Only a subset of the queries could be run as AWS Lambda currently supports a maximum container size of 1536 MB. The benchmark is designed to increase the output size by an order of magnitude for the a,b,c iterations. Given that the output size doesn't fit in Lambda memory, we currently can't process to compute the final output. 

```
|-----------------------|---------|---------|--------------|
| Technology            | Scan 1a | Scan 1b | Aggregate 2a | 
|-----------------------|---------|---------|--------------|
| Amazon Redshift (HDD) | 2.49    | 2.61    | 25.46        |
|-----------------------|---------|---------|--------------|
| Impala - Disk - 1.2.3 | 12.015  | 12.015  | 113.72       |
|-----------------------|---------|---------|--------------|
| Impala - Mem - 1.2.3  | 2.17    | 3.01    | 84.35        |
|-----------------------|---------|---------|--------------|
| Shark - Disk - 0.8.1  | 6.6     | 7       | 151.4        |
|-----------------------|---------|---------|--------------|
| Shark - Mem - 0.8.1   | 1.7     | 1.8     | 83.7         |
|-----------------------|---------|---------|--------------|
| Hive - 0.12 YARN      | 50.49   | 59.93   | 730.62       |
|-----------------------|---------|---------|--------------|
| Tez - 0.2.0           | 28.22   | 36.35   | 377.48       |
|-----------------------|---------|---------|--------------|
| BigLambda             | 39      | 47      | 200          |   
|-----------------------|---------|---------|--------------|

BigLambda Cost:

|---------|---------|--------------|
| Scan 1a | Scan 1b | Aggregate 2a | 
|---------|---------|--------------|
| 0.00477 | 0.0055  | 0.1129       |   
|---------|---------|--------------|
```

## Getting started
Here's how you get started with BigLambda.

### Prerequisites

* Lambda execution role with 
    * [S3 read/write access](http://docs.aws.amazon.com/lambda/latest/dg/with-s3-example-create-iam-role.html)
    * Cloudwatch log access (logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents)
 
Check policy.json for a sample that you can extend.

* To execute the driver locally, make sure you configure your AWS profile with access to: 
    * [S3](http://docs.aws.amazon.com/AmazonS3/latest/dev/example-policies-s3.html)
    * [Lambda](http://docs.aws.amazon.com/lambda/latest/dg/lambda-api-permissions-ref.html)

### Setting up a job

```

{
    "bucket": "big-data-benchmark",
        "prefix": "pavlo/text/1node/uservisits/",
        "jobBucket": "smallya-useast-1",
        "concurrentLambdas": 100,
        "mapper": {
            "name": "mapper.py",
            "handler": "mapper.lambda_handler",
            "zip": "mapper.zip"
        },
        "reducer":{
            "name": "reducer.py",
            "handler": "reducer.lambda_handler",
            "zip": "reducer.zip"
        },
        "reducerCoordinator":{
            "name": "reducerCoordinator.py",
            "handler": "reducerCoordinator.lambda_handler",
            "zip": "reducerCoordinator.zip"
        },
}

```

### Running an example

* Edit the configuration JSON 
* Write your mapper and reducer logic
* Run the driver:
 
	$ python driver.py

### Outputs 

```
smallya$ aws s3 ls s3://JobBucket/ biglambda-1node-0 --recursive --human-readable --summarize

2016-09-26 15:01:17   69 Bytes 0py-biglambda-1node-2/jobdata
2016-09-26 15:02:04   74 Bytes 0py-biglambda-1node-2/reducerstate.1
2016-09-26 15:03:21   51.6 MiB 0py-biglambda-1node-2/result 
2016-09-26 15:01:46   18.8 MiB 0py-biglambda-1node-2/task/
….

smallya$ head –n 3 result 
67.23.87,5.874290244999999
30.94.22,96.25011190570001
25.77.91,14.262780186000002
```
