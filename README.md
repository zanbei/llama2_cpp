# Optimize llama2 by llama.cpp, llama-cpp-python and Sagemaker deployment
## llama.cpp repo  
https://github.com/ggerganov/llama.cpp

## llama-cpp-python
https://github.com/abetlen/llama-cpp-python

## Quick start  
1. Download llama2 optimization model  (have to install git lfs firstly)

```bash
git lfs install
```
```bash
# Notice: This Step will download several models, you can mannually download single model

# git clone https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML
git clone https://huggingface.co/TheBloke/Llama-2-13B-chat-GGML
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
image_uri = 'xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/llama2-7b-13b-cpp-python-sagemaker'
role = get_execution_role()
# This can be dummy model file
model_dir = 's3://dummy/model.tar.gz'

# Create the SageMaker model instance
model = Model(
    image_uri=image_uri,
    role=role,
    model_data=model_dir
)

model.deploy(
    instance_type='ml.g4dn.xlarge',
    initial_instance_count=1,
    endpoint_name = 'llama2-cpp-test',
)
```
6. Invoke Sagemaekr Endpoint
```python
import boto3
import time

endpoint_name = 'llama2-cpp-test'
data = {"prompt": "hello world"}
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
# Parameters
* Model Deploy(you can modify in environment dict when deploy or modify source code in handler.py directly)
    1. CONTEXT_SIZE: Maximum context size
* Endpoint invoke(you can add it in input data dict when invoke endpoint)  
    1. max_tokens
    2. temperature


# Speed Test
* Instance type: ml.g4dn.xlarge
* Insance count: 1
* Task type: Chat and Generate
* Prompt: 'Q: I liked "Breaking Bad" and "Better call Saul". Do you have any recommendations of other shows I might like?\n A: '

* Result  

  | Option                  | llama2-7b-chat-q4_1           | llama2-13b-chat-q4_1 |  llama2-7b-chat-hf-native      |
  |-------------------------|------------------------|---------|------------|
  | Response Text     | ‘ Absolutely! If you enjoyed "Breaking Bad" and "Better Call Saul," here are some other great TV shows you might enjoy:1. "The Sopranos": This HBO series is a classic crime drama that follows the life of Tony Soprano, a New Jersey mob boss who seeks therapy to deal with his anxiety attacks. The show features a talented cast, including James Gandolfini and Edie Falco, and explores themes of loyalty, family, and the blurred lines between good and evil.2. "The Wire": This HBO series is often considered one of the greatest TV shows of all time. It's a gritty, realistic portrayal of the drug trade in Baltimore, exploring themes of crime, corruption, and the failing American Dream. The show features an incredible cast, including Dominic West and Idris Elba, and is known for its complex characters, intricate plotting, and thought-provoking themes.3. "Narcos": This Netflix series tells the true story of Pablo Escobar, the infamous Colombian drug lord, and the DEA agents who hunted him down. The show features stunning performances from Wagner Moura and Boyd Holbrook, and explores themes of power, corruption, and the war on drugs.4. "Peaky Blinders": This BBC series is set in post-World War I England and follows the exploits of the Shelby crime family. The show features a talented cast, including Cillian Murphy and Helen McCrory, and explores themes of loyalty, family, and power in a world torn apart by conflict and social change.5. "Sons of Anarchy": This FX series follows the lives of a close-knit outlaw motorcycle club in California, as they navigate the dangerous world of organized crime. The show features a talented cast, including Charlie Hunnam and Katey Sagal, and explores themes of loyalty, family, and redemption.6. "The Shield": This FX series follows the story of Detective Vic Mackey’|   ’If you enjoyed "Breaking Bad" and "Better Call Saul," here are some TV shows that you may also enjoy: 1. "The Sopranos": This classic HBO series explores the life of a New Jersey mob boss, Tony Soprano, as he navigates the criminal underworld and deals with personal and family issues. 2. "Narcos": This Netflix series tells the true story of Pablo Escobar, the infamous Colombian drug lord, and the DEA agents who hunt him down. 3. "Peaky Blinders": Set in post-World War I England, this BBC series follows a gangster family as they rise to power in the criminal underworld. 4. "Sons of Anarchy": This FX series explores the lives of a motorcycle club involved in gun-running and drug trafficking, while also delving into the personal struggles of its members. 5. "The Wire": This critically acclaimed HBO series examines the drug trade in Baltimore from multiple perspectives, including law enforcement, drug dealers, and politicians. 6. "Ozark": In this Netflix series, a financial advisor launders money for a drug cartel and relocates his family to the Missouri Ozarks, where he becomes embroiled in a local criminal underworld. 7. "Banshee": This Cinemax series follows an ex-con who assumes the identity of a small-town sheriff in order to hide from the gangster he robbed. 8. "True Detective": This anthology series features a different cast and storyline each season, but they all explore themes of crime and redemption. 9. "The Shield": This FX series follows a corrupt police detective and his team as they navigate the dangerous streets of Los Angeles. 10. "Orange is the New Black": Set in a women's prison, this Netflix series explores the lives of a diverse group of inmates and the challenges they face behind bars. I hope you find something that interests you in this list!’|  ‘I also like psychological thrillers and crime dramas. Answer: Oh, absolutely! If you enjoyed "Breaking Bad" and "Better Call Saul," here are some other shows you might like: 1. "The Sopranos" - This classic HBO series is a crime drama that follows the life of Tony Soprano, a mob boss struggling with family, business, and personal issues. 2. "Narcos" - This Netflix series tells the true story of Pablo Escobar, the infamous Colombian drug lord, and the efforts of law enforcement to bring him down. 3. "The Shield" - Also from FX, "The Shield" is a gritty crime drama that follows a corrupt cop’ |
  | Average Time Cost  | 13.40s                |   21.86s | 15.22s   |
  | Average Output Chars Length     | 1821 |      1791 |    713  |
  | Speed     | 135.86 Chars/s       |   81.93 Chars/s |   46.82 chars/s   |


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