# Optimize llama2 by llama.cpp, llama-cpp-python and Sagemaker deployment
## llama.cpp repo  
https://github.com/ggerganov/llama.cpp

## llama-cpp-python
https://github.com/abetlen/llama-cpp-python

## Quick start  
1. Download llama2 optimization model  (have to install git lfs firstly)

<!-- ```bash
git lfs install
``` -->
```bash
# Notice: This Step will download several models, you can mannually download single model

# git clone https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML
# git clone https://huggingface.co/TheBloke/Llama-2-13B-chat-GGML
wget https://huggingface.co/TheBloke/Firefly-Llama2-13B-v1.2-GGUF/resolve/main/firefly-llama2-13b-v1.2.Q2_K.gguf
```

2. Notice: you can modify Dockfile to ADD different model weights.

3. Build docker image
```bash
docker build -t llama2-13b-cpp-python-sagemaker .
```
4. Push the local image to AWS ECR Repo   
**Notice: replace region and account id, this example region=us-east-1**
```bash
# AWS ECR Login,
docker login -u AWS -p $(aws ecr get-login-password --region us-east-1) https://xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com

# Create ECR Repo
aws ecr create-repository --repository-name llama2-7b-13b-cpp-python-sagemaker --image-scanning-configuration scanOnPush=true --image-tag-mutability MUTABLE

# Tag Image
docker tag llama2-7b-13b-cpp-python-sagemaker:latest xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/llama2-7b-13b-cpp-python-sagemaker:latest

#Push Image to ECR
docker push xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/llama2-7b-13b-cpp-python-sagemaker:latest
```
5. Deploy Sagemaker Endpoint(Execute this python code snippet in juypter notebook or ec2)
```python
from sagemaker import get_execution_role
from sagemaker.model import Model
from sagemaker.predictor import Predictor
from sagemaker.session import Session

#ECR URI
image_uri = '969422986683.dkr.ecr.cn-northwest-1.amazonaws.com.cn/llama213bint4'
role = 'arn:aws-cn:iam::969422986683:role/AmazonSageMaker-ExecutionRole-20200517T121567'
# This can be dummy model file
model_dir = 's3://sagemaker-cn-northwest-1-969422986683/model.tar.gz'

# Create the SageMaker model instance
model = Model(
    image_uri=image_uri,
    role=role,
    model_data=model_dir
)

model.deploy(
    instance_type='ml.p3.2xlarge',
    initial_instance_count=1,
    endpoint_name = 'llama2-cpp-test',
)
```
6. Invoke Sagemaekr Endpoint
```python
import boto3
import time
import json

endpoint_name = 'llama2-cpp-test'
data = {"prompt": "how to learn english?"}
runtime_sagemaker_client = boto3.client(service_name="sagemaker-runtime")


body = json.dumps(data)

start = time.time()
response = runtime_sagemaker_client.invoke_endpoint(
    EndpointName = endpoint_name,
    ContentType  = "application/json",
    Body= body)

cost = time.time() - start     
result = response['Body'].read().decode('utf-8')

print('Response: ', result)
print("Cost Time:  %s seconds" % (cost))
print('Output Chars :', len(result))
print('Speed: {:.2f} Chars/s'.format(len(result)/float(cost)))
```


# Speed Test
* Instance type: ml.p3.2xlarge
* Insance count: 1
* Task type: Chat and Generate
* Prompt: 'how to learn english?'

* Result  
Response:  

learning english can be challenging, but it is an essential skill that will help you in your personal and professional life. here are some tips on how to learn english:

1. immerse yourself in the language - listen to english music, watch english movies or tv shows, read english books, and practice speaking with native speakers.

2. take a course - there are many online courses available that can help you improve your english skills. some popular platforms include duolingo, rosetta stone, and babbel.

3. practice, practice, practice - the more you practice speaking and writing in english, the better you will become. try to speak with native speakers as much as possible.

4. read regularly - reading in english can help improve your vocabulary and comprehension skills. choose books that are at your level so you don't get frustrated.

5. use resources online - there are many websites and apps available that can help you learn english, such as grammarly, linguee, and fluentu.

remember, learning a language takes time and effort, but with dedication and practice,
Cost Time:  5.032215595245361 seconds
Output Chars : 1076
Speed: 213.82 Chars/s

# Model Size
| Model  | Original Size   | Quantized Size(4-bit) |
|-------|-------|------|
|7B	    |13 GB	|3.9GB|
|13B	|24GB	|7.8GB|
|70B	|120GB	|38.5GB|

# llama-cpp-python API 
If you want to dive deep in deploy and invoke llama.cpp model, please ref link below:  
https://llama-cpp-python.readthedocs.io/en/latest/api-reference/  

# Verified Region
* us-east-1

# TODO
    1. llama2-70b model optimize