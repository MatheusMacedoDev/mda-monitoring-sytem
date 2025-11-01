# Deploy Python Lambda Function
## Files
- **lambda_function.py** -> File with a handle function to work as entrypoint for Lambda invocation
- **requirements.txt** -> File with Python external dependencies
- **Dockerfile** -> File with instruction for Docker build the Lambda

### Docker file example
```docker 
FROM public.ecr.aws/lambda/python:3.12

# Copy requirements.txt
COPY requirements.txt ${LAMBDA_TASK_ROOT}

# Install the specified packages
RUN pip install -r requirements.txt

# Copy function code
COPY lambda_function.py ${LAMBDA_TASK_ROOT}

# Set the CMD to your handler
CMD [ "lambda_function.handler" ]
```

## Deploy Process for the first time
1. Build the Docker image
```
docker buildx build --platform linux/amd64 --provenance=false -t youtube-collector:v0.0.1 .
```
2. Run the get-login-password command to authenticate the Docker CLI to your Amazon ECR registry
```
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <AWS account ID>.dkr.ecr.us-east-1.amazonaws.com
```
3. Create a repository in Amazon ECR using the create-repository command.
```
aws ecr create-repository --repository-name youtube-collector --region us-east-1 --image-scanning-configuration scanOnPush=true --image-tag-mutability MUTABLE
```
4. Run the docker tag command to tag your local image into your Amazon ECR repository as the latest version.
```
docker tag youtube-collector:v0.0.1 <AWS account ID>.dkr.ecr.us-east-1.amazonaws.com/youtube-collector:latest
```
5. Run the docker push command to deploy your local image to the Amazon ECR repository.
```
docker push <AWS account ID>.dkr.ecr.us-east-1.amazonaws.com/youtube-collector:latest
```
6. Create an execution role for the function
```
aws iam create-role \
  --role-name lambda-collector \
  --assume-role-policy-document '{"Version": "2012-10-17", "Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
```
7. Create the Lambda function.
```
aws lambda create-function \
  --function-name youtube-collector \
  --package-type Image \
  --code ImageUri=<AWS account ID>.dkr.ecr.us-east-1.amazonaws.com/youtube-collector:latest \
  --role arn:aws:iam::<AWS account ID>:role/lambda-collector
```
## Test
### Local
1. Start the Docker image with the **docker run** command.
```
docker run --platform linux/amd64 -p 9000:8080 youtube-collector:latest
```
2. Run the following curl command:
```
curl "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'
```
3. Get the container ID.
```
docker ps
```
4. Use the docker kill command to stop the container.
```
docker kill 3766c4ab331c
```
### Deploy
```
aws lambda invoke --function-name youtube-collector response.json
```
