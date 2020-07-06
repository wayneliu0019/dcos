# 基于arm64的DC/OS构建

## 编译环境
与DC/OS开发环境要求相同，见下链接
https://github.com/wayneliu0019/dcos#development-environment

唯一不同点在于底层OS必须是`ARM64`系统(推荐centos或者ubuntu系统)

这里以centos7环境为例，其基本安装配置步骤如下
1. 主机至少4GB内存，20GB可用磁盘
2. 更新系统 yum -y update
3. 主机安装git(>1.8.5), docker(>1.11), pyenv
4. 安装相关工具软件和其他依赖包
```
yum install -y tar unzip make
yum install readline readline-devel readline-static -y
yum install openssl openssl-static sqlite-devel -y
yum install bzip2-devel bzip2-libs -y
yum -y install gcc
yum -y install gcc-c++
yum install build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev xz-utils liblzma-dev python3-venv -y
```
5. git clone dcos源码到本地
6. 切换到指定分支或者tag，也可以根据tag创建新的本地分支
```
git branch <new-branch-name> <tag-name>
```
7. 切换到新建分支，用pyenv构建python3.6（3.5也可以）虚拟环境，并激活进入该编译环境
8. 调整/root/dcos/build_local.sh中的参数设置
```
sed -i s/python3.5/python/g /root/dcos/build_local.sh
```
其实就是将/root/dcosbuild_local.sh中的: 
```
python3.5 -m venv /tmp/dcos_build_venv
```
改成:
```
python -m venv /tmp/dcos_build_venv
```
修改完后,在git中确认完成本地提交(不用提交到远程repo)
```
git add build_local.sh
git commit -m "local modify"
```

至此，编译环境准备完毕。


## 针对ARM64的更改
首先确保能够在X86_64环境下可以正常编译OSS DC/OS，主要用以熟悉DC/OS的整个编译过程。

默认情况下的DC/OS是无法在ARM64环境下编译构建的，毕竟底层架构不同导致一些依赖包不兼容，需要做更改或者适配。

DC/OS对于ARM64系统的支持所做的更改有如下几个：
* Docker镜像

编译过程中用到了大量的docker镜像，需要更改为AMR64格式的，如将“alpine:3.4”更改为“arm64v8/alpine:3.5”

否则默认的基于x86_64格式的镜像不能正确在ARM系统上运行，绝大部分主流镜像在DockerHub上都有ARM64版本的。

这部分更改比较普遍，大量存在于各个组件的buildinfo.json以及pkgpanda组件中。

* ARM64格式的系统依赖文件

编译过程中一些组件会用到一些系统文件（如lib，so等），需要更改为使用ARM64的(将x86_64替换为aarch64)，这部分组件主要有2个：`mesos`和`adminrouter`，更改文件如下：
```
packages/adminrouter/build
packages/mesos/build
```

* ARM64格式的第三方包

一些组件在编译过程中会用到第三方软件包，如java，glide，golang等，默认等这些包都是x86_64格式的，无法直接用于ARM64环境下的DC/OS编译，这些包需要替换为ARM64格式的，相关组件和包如下：

**gen/build_deploy/bash/Dockerfile.in**
>glibc

**packages/dcos-cni/build**
>glide

**packages/java/buildinfo.json**
>java

**packages/strace/buildinfo.json**
>strace

**pkgpanda/docker/dcos-builder/Dockerfile**
>golang

***注意***
1. 上述第三方依赖包有的官方提供有ARM64版本的，直接更改链接即可，如golang，glide；有的官方未提供的，已经被放置在S3中，如java，glibc等；还有少部分是单独编译出来的，也放在S3中，如strace。
2. 不是所有第三方依赖包都需要提供针对ARM64版本的。有些不需要，比如exhibitor组件，exhibitor获取到指定版本的`zookeeper`源代码，然后重新编译，生成最终的安装包。这种情况就不用做任何更改，因为是由源码在ARM环境中编译出来的。
同样对于adminrouter组件中用到的`openresty`也是这种情况。



* 其他编译错误

DC/OS不能保证指定版本在任何时候都可以编译成功，尤其是针对一些老的版本。错误原因包括但不限于：
1. 编译过程中会用到一些未指定版本的包，使用当前最新包可能会存在兼容问题。
2. 一些组件用到的url地址已经不存在。
```
https://github.com/wayneliu0019/dcos/commit/4a1279865e67a0da41e79019d6741b13fc7a626b#diff-f15fbe7a4a73549d87cfc95e3acfbdd6R18
```
3. 部分组件参数不合适导致编译失败。
```
https://github.com/wayneliu0019/dcos/commit/4a1279865e67a0da41e79019d6741b13fc7a626b#diff-f15fbe7a4a73549d87cfc95e3acfbdd6R98

packages/adminrouter/extra/src/includes/http/agent.conf
packages/adminrouter/extra/src/includes/http/master.conf

https://github.com/wayneliu0019/dcos/commit/4a1279865e67a0da41e79019d6741b13fc7a626b#diff-b0a8c1157a2f5a0db8128fdb88511291R4

https://github.com/wayneliu0019/dcos/commit/89df99dc16d23423a7dafe5d833e4f0a3c385561

https://github.com/wayneliu0019/dcos/commit/d16a557a20cf1806640e4ad915cc25c15559eccb

```


## 当前支持的ARM64系统的DC/OS版本
当前提供如下支持ARM764的DC/OS版本：
```
1.11.0
1.11.9
1.12.4
1.12.5
```

分别对应如下代码分支：
```
arm64-1.11.0
arm64-1.11.9
arm64-1.12.4
arm64-1.12.5
```




## Troubleshooting
* **通过pyenv构建编译环境，编译早期版本的dcos1.11.0时候，提示urllib3版本错误**
```
Traceback (most recent call last):
File "/tmp/dcos_build_venv/lib/python3.5/site-packages/pkg_resources/__init__.py", line 651, in _build_master
ws.require(__requires__)
File "/tmp/dcos_build_venv/lib/python3.5/site-packages/pkg_resources/__init__.py",
line 952, in require
needed = self.resolve(parse_requirements(requirements))
File "/tmp/dcos_build_venv/lib/python3.5/site-packages/pkg_resources/__init__.py", line 844, in resolve
raise VersionConflict(dist, req).with_context(dependent_req) pkg_resources.ContextualVersionConflict: (urllib3 1.25.3 (/tmp/dcos_build_venv/lib/python3.5/site-packages), Requirement.parse('urllib3<1.23,>=1.21.1'), {'requests'})
```
**原因**

在dcos开始编译之前，会在当前python环境中下载必要的包，这些包定义在setup.py中，但很多包没有指定版本，如urllib3，那么就会下载当前最新的，一些组件如python-request在dcos1.12中要求的urllib3版本为1.25.3，但在dcos1.11.0中要求的urllib3版本为1.22。那么现在再次编译DC/OS 1.11.0版 本，可能会由于setup.py中没指定urllib3版本，导致出现上 的依赖版本错误。

**解决办法**

在重新编译dcos1.11.0版本时候，更改setup.py，指定urllib3包的版本， 如:
'urllib3==1.22'



* **git版本过低，出现不支持的命令**

该问题一般出现在ARM编译环境中，系统repo中提供的Git版本低于1.8.5，导致出现部分命令参数不兼容，如`git -C`


**解决办法**

安装1.8.5以上版本的git，如果没有源，就需要从源代码安装

git源码： 
https://mirrors.edge.kernel.org/pub/software/scm/git/

安装步骤：
https://gist.github.com/MHMDhub/d2d1a857fc5af5b18d6fff70fb4489b5


* **mesos编译报错“ g++: internal compiler error: Killed (program cc1plus)”**
```
......
g++: internal compiler error: Killed (program cc1plus)
Please submit a full bug report,
with preprocessed source if appropriate.
```
出现g++: internal compiler error: Killed (program cc1plus)错误，一般是由于内存或者swap空间不足导致的


**解决办法**

增大物理机内存，或者创建一个swap file，开启swap space，具体步骤参考如下连接：

https://www.oipapio.com/question-3763265

* **tar.gz文件无法识别(编译ARM64环境的1.12.5版本出错)**

编译完单个项 后，会在[dcos源码]/packages/cache/packages/下生成每个项目的tar.xz压缩文件，最终pkgpanda会将所有项 目拆包，然后再打包到一起，生成一个安装包。但此时报错如下
```
During handling of the above exception, another exception occurred: Traceback (most recent call last):
File "/tmp/dcos_build_venv/bin/release", line 11, in <module> load_entry_point('dcos-image', 'console_scripts', 'release')()
File "/root/dcos/release/__init__.py", line 937, in main release_manager.create('testing', options.channel, options.tag, variants)
File "/root/dcos/release/__init__.py", line 820, in create self.__config['options']['cloudformation_s3_url'] + '/' + repository_path,
tree_variants)
File "/root/dcos/release/__init__.py", line 328, in make_stable_artifacts
all_completes = do_build_packages(cache_repository_url, tree_variants) File "/root/dcos/release/__init__.py", line 578, in do_build_packages
result = pkgpanda.build.build_tree(package_store, True, tree_variants) File "/root/dcos/pkgpanda/build/__init__.py", line 753, in build_tree
'bootstrap': make_bootstrap(package_set),
File "/root/dcos/pkgpanda/build/__init__.py", line 744, in make_bootstrap
package_set.variant)
File "/root/dcos/pkgpanda/build/__init__.py", line 594, in
make_bootstrap_tarball repository.add(local_fetcher, pkg_id, False)
File "/root/dcos/pkgpanda/__init__.py", line 490, in add fetcher(id, tmp_path)
File "/root/dcos/pkgpanda/build/__init__.py", line 593, in local_fetcher shutil.unpack_archive(pkg_path, target, "gztar")
File "/root/.pyenv/versions/3.6.3/lib/python3.6/shutil.py", line 968, in unpack_archive
 
func(filename, extract_dir, **dict(format_info[2]))
File "/root/.pyenv/versions/3.6.3/lib/python3.6/shutil.py", line 913, in
_unpack_tarfile
"%s is not a compressed or uncompressed tar file" % filename)
shutil.ReadError: /root/dcos/packages/cache/packages/boto/boto- -785a6a0209448e13a85177a87d829e3a3892244e.tar.xz is not a compressed or uncompressed tar file
```

所有项目的tar.xz包生成没问题，但在最后通过python的 tarfile.open()打开时候报错，这些包可以通过tar命令手动拆包，而且通过手动tar命令打的tar.gz包也可以被代码识别运行，猜测可能是python中tar命令用法有问题。

**解决办法**

更改
https://github.com/wayneliu0019/dcos/blob/1.12.5/pkgpanda/util.py#L388
将生成tar.xz的命令中的`'-cJf'` , 改为`'cvf'`即可。

* **安装docker报错**

该问题一般出现在rhel编译环境中。
```
[root@ip-172-31-3-54 ~]#  yum install docker-ce-18.06.3.ce-3.el7 
。。。。。
--> Finished Dependency Resolution
Error: Package: docker-ce-18.06.3.ce-3.el7.aarch64 (docker-ce-stable)
           Requires: container-selinux >= 2.9
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest

```

**解决办法**

如下方式安装指定版本的container-seliux （再高版本的container-selinux会报错）
```
yum install http://mirror.centos.org/altarch/7/extras/aarch64/Packages/container-selinux-2.68-1.el7.noarch.rpm
```
再次安装docker成功

* **依赖包缺失**
```
gcc -pthread -Wno-unused-result -Wsign-compare -Wunreachable-code -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -fPIC -DUSE__THREAD -DHAVE_SYNC_SYNCHRONIZE -I/usr/include/ffi -I/usr/include/libffi -I/tmp/dcos_build_venv/include -I/root/.pyenv/versions/3.5.0/include/python3.5m -c c/_cffi_backend.c -o build/temp.linux-aarch64-3.5/c/_cffi_backend.o
      c/_cffi_backend.c:15:10: fatal error: ffi.h: No such file or directory
       #include <ffi.h>
                ^~~~~~~
      compilation terminated.
      error: command 'gcc' failed with exit status 1

```
**解决办法**

安装libffi依赖包
```
centos： yum install libffi-devel
ubuntu：apt-get install libffi-dev 

```



