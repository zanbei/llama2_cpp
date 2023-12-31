ARG CUDA_IMAGE="12.1.1-devel-ubuntu22.04"
FROM nvidia/cuda:${CUDA_IMAGE}

ENV CUDA_DOCKER_ARCH=all
ENV LLAMA_CUBLAS=1

# Install the package
RUN apt update && apt install -y python3 python3-pip
RUN python3 -m pip install --upgrade pip pytest cmake scikit-build setuptools fastapi uvicorn sse-starlette pydantic-settings

ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get  -y -f install build-essential libgeos-dev

ARG FUNCTION_DIR="/opt/program"
WORKDIR ${FUNCTION_DIR}

#ADD Python file
ADD requirements.txt ${FUNCTION_DIR}
RUN pip3 install -r ${FUNCTION_DIR}/requirements.txt
RUN LLAMA_CUBLAS=1 pip install llama-cpp-python
RUN pip3 install ctransformers[cuda]>=0.2.24
#ADD model
RUN mkdir -p ${FUNCTION_DIR}/model

# ----- Select 7B or 13B Model-----
# ADD llama-2-7b-chat.ggmlv3.q4_K_M.bin ${FUNCTION_DIR}/model
# ENV MODEL_PATH="${FUNCTION_DIR}/model/llama-2-7b-chat.ggmlv3.q4_K_M.bin"

ADD firefly-llama2-13b-v1.2.Q2_K.gguf ${FUNCTION_DIR}/model
ENV MODEL_PATH="${FUNCTION_DIR}/model/firefly-llama2-13b-v1.2.Q2_K.gguf"

# ---------------------------------

# Add Python Files
ADD handler.py ${FUNCTION_DIR}
ADD sm_llm.py ${FUNCTION_DIR}


ENV PYTHONUNBUFFERED=TRUE
ENV PYTHONDONTWRITEBYTECODE=TRUE
ENV PYTHONIOENCODING="utf8"

WORKDIR ${FUNCTION_DIR}
ENTRYPOINT ["python3", "sm_llm.py"]