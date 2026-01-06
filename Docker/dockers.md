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
