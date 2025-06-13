# AWS commands used in this blog

## initial configuration
```
aws configure
AWS Access Key ID [None]: <service_key.bucket>
Secret Access Key [None]: 
Default region name [None]:  
Default output format [None]:
```
## verify the configuration and login
```
aws s3api get-bucket-location --bucket <service_key.bucket>
```

## push recursively the whole content of the current folder to the specified bucket
```
aws s3 cp –recursive . s3://<service_key.bucket>/ade-v2/eval/
```

## list the files stored in the specific folder, recursively
```
aws s3 ls --recursive s3://<service_key.bucket>/ade-v2/eval/
```

## list the db files resulted by the execution of the configurations
```
aws s3 ls --recursive s3://<service_key.bucket>/output/  | grep db
```

## download all files resulted by the execution of the configurations
```
aws s3 cp --recursive s3://hcp-cc9c52a2-5f62-4563-bcc2-fdfb0d582164/output/
```

## remove all the content from the bucket
```
aws s3 rm --recursive s3://hcp-cc9c52a2-5f62-4563-bcc2-fdfb0d582164
```

## Remove all the output but keep the input files
```
aws s3 rm --recursive s3://hcp-cc9c52a2-5f62-4563-bcc2-fdfb0d582164/output/
```

## Remove all the runs but keep the test data
```
aws s3 rm --recursive s3://hcp-cc9c52a2-5f62-4563-bcc2-fdfb0d582164/ade-v2/eval-data/runs
```

## Delete only a specific run file
```
aws s3 rm --recursive s3://hcp-cc9c52a2-5f62-4563-bcc2-fdfb0d582164/ade-v2/eval-data/runs/gpt35turbo.json
```
