FROM alpine/git:2.36.2 as download

COPY clone.sh /clone.sh

RUN . /clone.sh taming-transformers https://github.com/CompVis/taming-transformers.git 24268930bf1dce879235a7fddd0b2355b84d7ea6 \
  && rm -rf data assets **/*.ipynb

RUN . /clone.sh stable-diffusion-stability-ai https://github.com/Stability-AI/stablediffusion.git 47b6b607fdd31875c9279cd2f4f16b92e4ea958e \
  && rm -rf assets data/**/*.png data/**/*.jpg data/**/*.gif

RUN . /clone.sh CodeFormer https://github.com/sczhou/CodeFormer.git c5b4593074ba6214284d6acd5f1719b6c5d739af \
  && rm -rf assets inputs

RUN . /clone.sh BLIP https://github.com/salesforce/BLIP.git 48211a1594f1321b00f14c9f7a5b4813144b2fb9
RUN . /clone.sh k-diffusion https://github.com/crowsonkb/k-diffusion.git 5b3af030dd83e0297272d861c19477735d0317ec
RUN . /clone.sh clip-interrogator https://github.com/pharmapsychotic/clip-interrogator 2486589f24165c8e3b303f84e9dbbea318df83e8


FROM alpine:3.17 as xformers
RUN apk add --no-cache aria2
RUN aria2c -x 5 --dir / --out wheel.whl 'https://github.com/AbdBarho/stable-diffusion-webui-docker/releases/download/5.0.0/xformers-0.0.17.dev449-cp310-cp310-manylinux2014_x86_64.whl'

#FROM python:3.10.9-slim
#FROM nvidia/cuda:12.1.0-devel-ubuntu18.04
FROM nvidia/cuda:11.7.0-devel-ubuntu18.04
#FROM qts8n/cuda-python:runtime

ENV DEBIAN_FRONTEND=noninteractive PIP_PREFER_BINARY=1

# for GroundingDINO
ENV AM_I_DOCKER=True
ENV BUILD_WITH_CUDA=True
ENV CUDA_HOME=/usr/local/cuda-11.7/
ENV TORCH_CUDA_ARCH_LIST="3.5;5.0;6.0;6.1;7.0;7.5;8.0;8.6+PTX"

RUN apt-get update && apt install software-properties-common -y && apt-get clean
RUN add-apt-repository ppa:deadsnakes/ppa
RUN apt-get update && apt-get install python3.10-dev python3.10-distutils curl libcairo2-dev -y && apt-get clean
RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.10 1
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
RUN python get-pip.py
RUN apt-get update && apt-get install -y locales && rm -rf /var/lib/apt/lists/* \
	&& localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install torch==1.13.1+cu117 torchvision --extra-index-url https://download.pytorch.org/whl/cu117

RUN apt-get update && apt install fonts-dejavu-core rsync git jq moreutils libgl1 libglib2.0-0 build-essential cargo rustc -y && apt-get clean

RUN --mount=type=cache,target=/root/.cache/pip \
  git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git && \
  cd stable-diffusion-webui && \
  git reset --hard d7aec59c4eb02f723b3d55c6f927a42e97acd679 && \
  pip install -r requirements_versions.txt

RUN --mount=type=cache,target=/root/.cache/pip  \
  --mount=type=bind,from=xformers,source=/wheel.whl,target=/xformers-0.0.15-cp310-cp310-linux_x86_64.whl \
  pip install triton /xformers-0.0.15-cp310-cp310-linux_x86_64.whl

ENV ROOT=/stable-diffusion-webui


COPY --from=download /repositories/ ${ROOT}/repositories/
RUN mkdir ${ROOT}/interrogate && cp ${ROOT}/repositories/clip-interrogator/data/* ${ROOT}/interrogate
RUN --mount=type=cache,target=/root/.cache/pip \
  pip install -r ${ROOT}/repositories/CodeFormer/requirements.txt

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install pyngrok \
  git+https://github.com/TencentARC/GFPGAN.git@8d2447a2d918f8eba5a4a01463fd48e45126a379 \
  git+https://github.com/openai/CLIP.git@d50d76daa670286dd6cacf3bcd80b5e4823fc8e1 \
  git+https://github.com/mlfoundations/open_clip.git@bb6e834e9c70d9c27d0dc3ecedeebeaeb1ffad6b \
  pycairo \
  pycocotools

# GroundingDINO
RUN --mount=type=cache,target=/root/.cache/pip \
  git clone https://github.com/IDEA-Research/Grounded-Segment-Anything && \
  cd Grounded-Segment-Anything && \
  git reset --hard 2e02b7dbca6cb5f047f5df031eea424072911b1b && \
  python -m pip install --no-cache-dir -e GroundingDINO && \
  cd ..

# TODO: either remove if fixed in A1111 (unlikely) or move to the top with other apt stuff
RUN apt-get update && apt-get -y install libgoogle-perftools-dev && apt-get clean
ENV LD_PRELOAD=libtcmalloc.so

# Note: don't update the sha of previous versions because the install will take forever
# instead, update the repo state in a later step

ARG SHA=a9fed7c364061ae6efb37f797b6b522cb3cf7aa2
RUN --mount=type=cache,target=/root/.cache/pip \
  cd stable-diffusion-webui && \
  git fetch && \
  git pull && \
  pip install -r requirements_versions.txt
# war statt vorletze  git reset --hard ${SHA} && \

RUN --mount=type=cache,target=/root/.cache/pip  pip install -U opencv-python-headless

COPY . /docker

#RUN \
#  python3 /docker/info.py ${ROOT}/modules/ui.py && \
#  mv ${ROOT}/style.css ${ROOT}/user.css && \
#  # one of the ugliest hacks I ever wrote \
#  sed -i 's/in_app_dir = .*/in_app_dir = True/g' /usr/local/lib/python3.10/site-packages/gradio/routes.py
RUN git config --global --add safe.directory '*'

WORKDIR ${ROOT}
ENV NVIDIA_DRIVER_CAPABILITIES=compute,utility
ENV NVIDIA_VISIBLE_DEVICES=all
ENV CLI_ARGS=""

EXPOSE 7860
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD python -u webui.py --listen --port 7860 ${CLI_ARGS}