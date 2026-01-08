### 1.容器技术的基本概念
<img width="772" height="180" alt="image" src="https://github.com/user-attachments/assets/35890058-a3f6-4088-82f1-3d636ddbda10" />

容器和虚拟机具有相似的资源隔离和分配优势，但功能有所不同，因为容器虚拟化的是操作系统，而不是硬件，因此容器更容易移植，效率也更高
云服务提供商通常采用虚拟机技术隔离不同的用户。而 Docker 通常用于隔离不同的应用 ，例如前端，后端以及数据库
传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便

### 2.容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。前面讲过镜像使用的是分层存储，容器也是如此。

### 3.
<img width="818" height="815" alt="image" src="https://github.com/user-attachments/assets/120b9637-43ff-43b6-a688-415994cc9107" />

### 4.
<img width="827" height="521" alt="image" src="https://github.com/user-attachments/assets/f9a9d839-cde1-4660-bcbb-12105da810ec" />

### 5
docker build -f ./ Dockerfile  根据给出的Dockerfile构建镜像，Dockerfile文件中

### 6.上下文路径
上下文路径，是指 docker 在构建镜像，有时候想要使用到本机的文件（比如复制），docker build 命令得知这个路径后，会将路径下的所有内容打包。

解析：由于 docker 的运行模式是 C/S。我们本机是 C，docker 引擎是 S。实际的构建过程是在 docker 引擎下完成的，所以这个时候无法用到我们本机的文件。这就需要把我们本机的指定目录下的文件一起打包提供给 docker 引擎使用。

如果未说明最后一个参数，那么默认上下文路径就是 Dockerfile 所在的位置。

### 7
说白了怎么一步一步构建镜像就看dockerfile，而构建镜像的时候还要有本地程序代码
![Uploading image.png…]()
