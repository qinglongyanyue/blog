# Cornell大学derecho通信框架测试

## 项目地址
https://github.com/Derecho-Project/derecho-unified

目前是Cornell大学的研究项目，测试研究一下

### 安装依赖

建议使用ubuntu16.04来测试

- 安装GCC7 & cmake
```
udo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update 
sudo apt-get install gcc-7 g++-7

sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 100
sudo update-alternatives --config gcc

apt install cmake

sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 100
sudo update-alternatives --config g++

gcc -v
```

- 安装依赖RDMA相关依赖：
```
apt install librdmacm-dev
apt install libibverbs-dev
```

- 安装libboost
```
apt install libboost-dev libboost-system-dev
```

- 下载代码编译

```
git clone --recursive https://github.com/Derecho-Project/derecho-unified.git
mkdir Release
cd Release
cmake -DCMAKE_BUILD_TYPE=Release ..
make
```

- 假设2台VM测试，将编译好的包复制到2台VM中

- 创建配置文件（以TCP通信为例）
```
# vim rdma.cfg
provider = sockets
domain = eth0
rx_depth = 16
tx_depth = 16
```

- 跑一个简单的测试

```
VM1:
cd derecho-unified/Release/derecho-unified/experiments
./derecho_bw_test 10000 0 15 1000 0
Please enter '[node_rank] [num_nodes]': 0 2
Please enter IP Address for node 0: 192.168.0.143
Please enter IP Address for node 1: 192.168.0.181
```

```
VM2:
cd derecho-unified/Release/derecho-unified/experiments
./derecho_bw_test 10000 0 15 1000 0
Please enter '[node_rank] [num_nodes]': 1 2
Please enter IP Address for node 0: 192.168.0.143
Please enter IP Address for node 1: 192.168.0.181

# cat data_derecho_bw #查看结果
```
