# syntax=docker/dockerfile:1
FROM alpine/git:2.36.2 as download

FROM alpine:3 as xformers
RUN apk add aria2
RUN aria2c -x 5 --dir / --out wheel.whl 'https://gh.api.99988866.xyz/https://github.com/AbdBarho/stable-diffusion-webui-docker/releases/download/4.0.0/xformers-0.0.16rc393-cp310-cp310-manylinux2014_x86_64.whl'

FROM python:3.10-slim

SHELL ["/bin/bash", "-ceuxo", "pipefail"]

ENV DEBIAN_FRONTEND=noninteractive PIP_PREFER_BINARY=1 PIP_NO_CACHE_DIR=1

#RUN pip install torch==1.12.1+cu116 torchvision==0.13.1+cu116 --extra-index-url https://download.pytorch.org/whl/cu116
RUN pip install torch==1.12.1+cu116 torchvision==0.13.1+cu116 --extra-index-url https://download.pytorch.org/whl/cu116 -i https://pypi.tuna.tsinghua.edu.cn/simple

RUN  sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && apt-get clean
RUN apt-get update && apt install fonts-dejavu-core rsync git jq moreutils -y && apt-get clean


RUN <<EOF
git clone https://hub.njuu.cf/AUTOMATIC1111/stable-diffusion-webui.git
cd stable-diffusion-webui
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements_versions.txt
EOF

RUN --mount=type=cache,target=/root/.cache/pip  \
  --mount=type=bind,from=xformers,source=/wheel.whl,target=/xformers-0.0.15-cp310-cp310-linux_x86_64.whl \
  pip install triton /xformers-0.0.15-cp310-cp310-linux_x86_64.whl -i https://pypi.tuna.tsinghua.edu.cn/simple

ENV ROOT=/stable-diffusion-webui


COPY repositories /stable-diffusion-webui/repositories
COPY --from=download /git/ ${ROOT}
RUN mkdir ${ROOT}/interrogate && cp ${ROOT}/repositories/clip-interrogator/data/* ${ROOT}/interrogate
RUN --mount=type=cache,target=/root/.cache/pip \
  pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -r ${ROOT}/repositories/CodeFormer/requirements.txt

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install opencv-python-headless  -i https://pypi.tuna.tsinghua.edu.cn/simple\
  git+https://hub.njuu.cf/TencentARC/GFPGAN.git@8d2447a2d918f8eba5a4a01463fd48e45126a379 \
  git+https://hub.njuu.cf/openai/CLIP.git@d50d76daa670286dd6cacf3bcd80b5e4823fc8e1 \
  git+https://hub.njuu.cf/mlfoundations/open_clip.git@bb6e834e9c70d9c27d0dc3ecedeebeaeb1ffad6b \
  pyngrok

# Note: don't update the sha of previous versions because the install will take forever
# instead, update the repo state in a later step

ARG SHA=685f9631b56ff8bd43bce24ff5ce0f9a0e9af490
RUN --mount=type=cache,target=/root/.cache/pip <<EOF
cd stable-diffusion-webui
git fetch
git reset --hard ${SHA}
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements_versions.txt
EOF

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install -U opencv-python-headless transformers>=4.24 -i https://pypi.tuna.tsinghua.edu.cn/simple

COPY . /docker

RUN <<EOF
python3 /docker/info.py ${ROOT}/modules/ui.py
mv  ${ROOT}/style.css ${ROOT}/user.css
sed -i 's/os.rename(tmpdir, target_dir)/shutil.move(tmpdir,target_dir)/' ${ROOT}/modules/ui_extensions.py
EOF

WORKDIR ${ROOT}
ENV CLI_ARGS=""
EXPOSE 7860
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD python3 -u webui.py --listen --port 7860 ${CLI_ARGS}
