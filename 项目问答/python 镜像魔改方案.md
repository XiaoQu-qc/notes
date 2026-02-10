### 1. 使用python魔改版的背景
在构建服务的私有化镜像时，我们需要对镜像中的源代码进行加密，这样其他人无法从公开的容器运行环境中获取到项目源代码的信息。但是由于基础的python 镜像并没有给我们提供加密源码的脚本，也没有提供具有解密已加密源码功能的python解释器，因此我们需要在镜像构建时实现这两个部分。第一个部分是在构建项目的运行镜像时添加加密脚本并运行。第二个部分则需要对python源码进行修改。
### 2.魔改python源码
可以先去问问负责的同学代码仓库中有没有指定python版本的魔改版，比如说项目中要用到python3.10的基础镜像ai_app / common / python · GitLab，git仓库中有已经魔改过的python3.10的源码，那么第二步可以跳过，直接拿现成的用即可。
以photo-optimization项目为例，如果使用python3.10的基础镜像，会导致依赖的版本问题，相比于解决版本问题，换成python3.8作为基础镜像可能是更简单的解决方案。
#### 1. 下载源码
python/cpython at v3.8.10可以使用这个github地址，上面提到的python3.10代码仓库还是需要拉取到本地的，因为要参考里面的dockerfile，images和src下创建3.10.12的同级文件夹3.8，源码放入src/3.8
#### 2.写dockerfile
```
FROM hub-mirror.wps.cn/ai-app/cv/ubuntu20.04:python3.8 AS buildbase

ENV DEBIAN_FRONTEND=noninteractive

RUN apt update \
  && apt install -y gcc automake autoconf libtool make build-essential libpq-dev libreadline-dev \
  && apt install -y libssl-dev libpq5 libpq-dev libbz2-dev libgdbm-dev liblzma-dev libsqlite3-dev zlib1g-dev \
  && apt install -y tk-dev libffi-dev libgdbm-compat-dev vim gdb wget lsof htop libgl1-mesa-glx curl \
  && apt install -y cmake tzdata iputils-ping telnet net-tools tini dos2unix

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone

COPY src/3.8.10 /python_build/kna_python3
WORKDIR /python_build/kna_python3

RUN mkdir /kna_python3  \
  && chmod +x configure \
  && ./configure --enable-optimizations --enable-shared --with-openssl=/usr --prefix=/kna_python3  \
  && make -j  \
  && make install

FROM hub-mirror.wps.cn/ai-app/cv/ubuntu20.04:python3.8
COPY --from=buildbase /kna_python3 /kna_python3

ENV PATH="/kna_python3/bin:${PATH}"

RUN rm /usr/bin/python3 \
  && rm /usr/bin/python3.8 \
  && rm -rf /usr/bin/python \
  && rm /usr/local/bin/pip3 \
  && rm /usr/local/bin/pip \
  && ln -s /kna_python3/bin/python3 /usr/bin/python3 \
  && ln -s /kna_python3/bin/python3 /usr/bin/python3.8 \
  && ln -s /kna_python3/bin/python3 /usr/bin/python \
  && ln -s /kna_python3/bin/pip3 /usr/local/bin/pip3 \
  && ln -s /kna_python3/bin/pip3 /usr/local/bin/pip \
  && rm -rf /usr/lib/x86_64-linux-gnu/libpython3.8.so.1.0 && ln -s /kna_python3/lib/libpython3.8.so.1.0 /usr/lib/x86_64-linux-gnu/libpython3.8.so.1.0 \
  && rm -rf /usr/lib/x86_64-linux-gnu/libpython3.8.so && ln -s /kna_python3/lib/libpython3.8.so /usr/lib/x86_64-linux-gnu/libpython3.8.so
```
最后的RUN命令作用是彻底替换系统默认的 Python 3.8，强制让容器内所有程序使用我们从源码编译的/kna_python3下的 Python 3.8
#### 3. 魔改python源码
魔改python3.8.10源码 (33a5245f) · Commits · ai_app / common / python · GitLab
修复文件缺失问题 (bbf6cd9c) · Commits · ai_app / common / python · GitLab
参考这两个commit进行修改，如果是其他版本的python也是对这些地方修改，其中新增的decryptfileop.c就是实现对加密文件的解密。
#### 4.提醒
不同版本在魔改时可能会略有差异，要根据情况做出修改。拿魔改python3.8时来举例，在启动容器并运行.py文件时，发现.py文件并没有被正确的解密，python解释器无法运行加密的python文件导致报错，后来经过一系列的调试排查，发现在Python 启动入口Modules/main.c中_Py_wfopen(filename, L"rb")中并不调用解密函数decrypt_fopen()，需要替换为在相同位置python3.10使用的_Py_fopen_obj()，并且做一些适配性的修改
3.构建魔改python3.8的基础镜像
获取基础镜像产物hub-mirror.wps.cn/ai-app/ubuntu_crt_python:20260127-035912
4.将产物镜像作为基础镜像
```
FROM hub-mirror.wps.cn/ai-app/ubuntu_crt_python:20260127-035912

RUN apt-get update && apt-get install -y vim libopencv-dev cmake build-essential

COPY base_dockerfile/requirements.txt requirements.txt
COPY package/ /tmp/package/

RUN pip3 install --upgrade pip && \
    pip3 install --no-cache-dir setuptools==44.0.0 && \
    pip3 install --no-cache-dir -r requirements.txt -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple && \
    pip3 install --no-cache-dir --find-links /tmp/package /tmp/package/PdfCvCommonLib-1.1.39-py3-none-any.whl && \
    rm -rf /tmp/package在之后容器启动成功以后，我们在镜像中的python解释器就可以解释运行已加密的python文件。
```
#### 5.源码加密
前面说过第一个部分是在构建项目的运行镜像时添加加密脚本并运行，具体的加密逻辑在第7行，镜像构建时会运行一个加密脚本，加密指定路径下的所有源码。
```
# python代码加密,根据具体项目修改

FROM hub-mirror.wps.cn/ai-app/python-code-file-encrypt:20240725151812 AS codeencrypt

# COPY ./ai_app-cv-photo_optimization /src/app/photo_optimization 
# 平台路径
COPY src/ /src/app/photo_optimization 
RUN python3 /src/code_file_encrypt.py --code_root_path /src/app

#SEC_SCAN#/src/app

# 运行环境,基础镜像解释器魔改解密功能
FROM hub-mirror.wps.cn/ai-app-private/photo-optimization-crt:3.8.10-ubuntu-20260127-064037

WORKDIR /apps

#加密后的代码会放在/src/app/package下
COPY --from=codeencrypt /src/app/package/photo_optimization /apps/  

# 依赖于kae传入
ARG BuildCommit=
ENV PCV_COMMIT_ID ${BuildCommit}
ENV LD_LIBRARY_PATH /apps/iutils/lib

RUN pip3 install xmltodict prometheus_client redis==5.3.0 && \
    pip3 install --no-cache-dir "aiohttp==3.6.2" "async-timeout==3.0.1" && \
    pip3 install --no-cache-dir aiapp_datarush -i https://mirrors.wps.cn/pypi/simple --extra-index-url https://mirrors.wps.cn/pypi/dev && \
    cd /apps/cpp/cal_height_width && mkdir build && cd build && cmake .. && make && \
    chmod a+x /apps/compile_cy.sh && cd /apps && ./compile_cy.sh

CMD ["python3", "main.py"]
```
