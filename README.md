
# IAC for GYANT Challenge App

#Files 

01-ecr.yaml (This template provision a private container registry )
02-vpc.yaml (This template provision the Network for the environment)
03-iam.yaml (This template provision the roles necessaries for the ECS)
04-ecs.yaml (This template provision a ECS Cluster, Load Balancer and deploy the App)
05-ala.yaml (This template provision a SNS Topic and configure 4 Alarmes with % of HTTP Errors)
06-waf.yaml (This template provision a Web Waf)
Dockerfile  (DockerFile to Build the continer)
architecture.png (Graphic detail of the stack)



#Setup

##1- Clone the challenge repo and had the Dockerfile
```sh
[hrodrigues@zelda test]$ git clone https://github.com/GYANTINC/gyant-challenge-app.git
Cloning into 'gyant-challenge-app'...
remote: Enumerating objects: 46, done.
remote: Counting objects: 100% (46/46), done.
remote: Compressing objects: 100% (41/41), done.
remote: Total 46 (delta 2), reused 46 (delta 2), pack-reused 0
Receiving objects: 100% (46/46), 47.74 KiB | 575.00 KiB/s, done.
Resolving deltas: 100% (2/2), done.

[hrodrigues@zelda test]$ cp Dockerfile gyant-challenge-app/
```

##2- Create the Docker Registry
###2.1 - Create stack
```sh
[hrodrigues@zelda test]$ aws cloudformation create-stack --stack-name ecrrepo --template-body file://\$PWD/01-ecr.yaml 
{
    "StackId": "arn:aws:cloudformation:eu-west-1:534451283295:stack/ecrrepo/762360f0-6f7c-11eb-9320-0665d7300625"
}
```
###2.2 - Get URI
```sh
[hrodrigues@zelda test]$ aws cloudformation describe-stacks --stack-name ecrrepo --query "Stacks[0].Outputs[?OutputKey=='GyantincContainerUri'].OutputValue" --output text
534451283295.dkr.ecr.eu-west-1.amazonaws.com/gyant-challenge-app
```

###2.3- Authenticate in ECR, build and push image
```sh
[hrodrigues@zelda test]$ cd gyant-challenge-app/
[hrodrigues@zelda gyant-challenge-app]$ aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 534451283295.dkr.ecr.eu-west-1.amazonaws.com
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Login Succeeded!
[hrodrigues@zelda gyant-challenge-app]$ docker build -t gyant-challenge-app .
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
STEP 1: FROM node:13
STEP 2: RUN mkdir -p /home/node/app/node_modules && chown -R node:node /home/node/app
--> 99afbdb02b8
STEP 3: WORKDIR /home/node/app
--> 20c4e0db76b
STEP 4: COPY package*.json ./
--> 348dab5bbd1
STEP 5: RUN npm install

> bcrypt@4.0.1 install /home/node/app/node_modules/bcrypt
> node-pre-gyp install --fallback-to-build

node-pre-gyp WARN Using needle for node-pre-gyp https download 
node-pre-gyp WARN Pre-built binaries not installable for bcrypt@4.0.1 and node@13.14.0 (node-v79 ABI, glibc) (falling back to source compile with node-gyp) 
node-pre-gyp WARN Hit error Remote end closed socket abruptly. 
make: Entering directory '/home/node/app/node_modules/bcrypt/build'
  CC(target) Release/obj.target/nothing/../node-addon-api/src/nothing.o
  AR(target) Release/obj.target/../node-addon-api/src/nothing.a
  COPY Release/nothing.a
  CXX(target) Release/obj.target/bcrypt_lib/src/blowfish.o
  CXX(target) Release/obj.target/bcrypt_lib/src/bcrypt.o
  CXX(target) Release/obj.target/bcrypt_lib/src/bcrypt_node.o
  SOLINK_MODULE(target) Release/obj.target/bcrypt_lib.node
  COPY Release/bcrypt_lib.node
  COPY /home/node/app/node_modules/bcrypt/lib/binding/napi-v3/bcrypt_lib.node
  TOUCH Release/obj.target/action_after_build.stamp
make: Leaving directory '/home/node/app/node_modules/bcrypt/build'
npm WARN gyant-crud@1.0.0 No repository field.
npm WARN gyant-crud@1.0.0 No license field.

added 360 packages from 247 contributors and audited 360 packages in 13.838s

27 packages are looking for funding
  run `npm fund` for details

found 9 vulnerabilities (5 low, 2 moderate, 2 high)
  run `npm audit fix` to fix them, or `npm audit` for details
--> 1d8d5a0433d
STEP 6: COPY . .
--> a53d9db5e22
STEP 7: COPY --chown=node:node . .
--> bf2191a2ceb
STEP 8: USER node
--> 9a2c1cdc015
STEP 9: EXPOSE 3000
--> 1974308c144
STEP 10: CMD [ "npm", "start" ]
STEP 11: COMMIT gyant-challenge-app
--> 9abe8c0d8be
9abe8c0d8be88cd5383ea274b76a13f99adf4561cf3b87fae1b95ec54ac74006

[hrodrigues@zelda gyant-challenge-app]$ docker tag gyant-challenge-app:latest 534451283295.dkr.ecr.eu-west-1.amazonaws.com/gyant-challenge-app:latest
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.

[hrodrigues@zelda gyant-challenge-app]$ docker push 534451283295.dkr.ecr.eu-west-1.amazonaws.com/gyant-challenge-app:latest
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Getting image source signatures
Copying blob 5aea01ea0a0f done  
Copying blob 38c2f9ead82d done  
Copying blob 0dabcc98eeef done  
Copying blob c96f2308ab16 done  
Copying blob 6885f9305c0a done  
Copying blob 05f4935ad90a done  
Copying blob d8183b2c9c73 done  
Copying blob ee50c22fdf6c done  
Copying blob 93c79b339996 done  
Copying blob ed09928f5a32 done  
Copying blob 98ef813507fe done  
Copying blob b380e82ead5f done  
Copying blob 88c50b21ae61 done  
Copying blob d91ca53cbc59 done  
Copying config 9abe8c0d8b done  
Writing manifest to image destination
Storing signatures
```

##3- Create VPC
```sh
[hrodrigues@zelda test]$ aws cloudformation create-stack --stack-name vpc --template-body file://\$PWD/02-vpc.yaml 
{
    "StackId": "arn:aws:cloudformation:eu-west-1:534451283295:stack/vpc/a98e6a10-6f7d-11eb-8425-0264e01e178b"
}
```
4- Create IAM Roles
```sh
[hrodrigues@zelda test]$ aws cloudformation create-stack --stack-name iamroles --template-body file://\$PWD/03-iam.yaml --capabilities CAPABILITY_IAM
{
    "StackId": "arn:aws:cloudformation:eu-west-1:534451283295:stack/iamroles/2b7459e0-6f7e-11eb-9243-06e31ff0cd99"
}
```
5- Create ECS Stack
```sh
[hrodrigues@zelda test]$ aws cloudformation create-stack --stack-name ecsapp --template-body file://\$PWD/04-ecs.yaml
{
    "StackId": "arn:aws:cloudformation:eu-west-1:534451283295:stack/ecsapp/78b67d00-6f7e-11eb-9ce7-06e549b26cbb"
}
```
5.1 GET URL
```sh
[hrodrigues@zelda test]$ aws cloudformation describe-stacks --stack-name ecsapp --query "Stacks[0].Outputs[?OutputKey=='UrlEndpoint'].OutputValue" --output text
http://gyantinc-services-1667129062.eu-west-1.elb.amazonaws.com
```
6- Configure CloudWatch Alarmes
```sh
[hrodrigues@zelda test]$ aws cloudformation create-stack --stack-name alarm --template-body file://\$PWD/05-alarm.yaml --parameters ParameterKey=MailingListEmail,ParameterValue=yugomail@gmail.com
{
    "StackId": "arn:aws:cloudformation:eu-west-1:534451283295:stack/alarm/18e56da0-6f83-11eb-8285-06282834de15"
}
```
7- Create Waf stack
```sh
[hrodrigues@zelda test]$ aws cloudformation create-stack --stack-name waf --template-body file://\$PWD/06-waf.yaml --parameters ParameterKey=IpRange,ParameterValue=217.129.241.137/32
{
    "StackId": "arn:aws:cloudformation:eu-west-1:534451283295:stack/waf/a0551ce0-6f88-11eb-9321-0a86526df8eb"
}

```



