# How to Run Fn-Project on a Raspberry Pi
My attempt to run fnproject on a RaspberryPi.

Inspired from <https://medium.com/@varpa89/run-fn-project-on-your-raspberry-pi-fa17f5067b47>

## Install Golang on RaspberryPi
### Install go - from binary

```bash
$ wget https://dl.google.com/go/go1.10.2.linux-armv6l.tar.gz
$ sudo tar -C /usr/local -xvf go1.10.2.linux-armv6l.tar.gz
$ cat >> ~/.bashrc << 'EOF'
export GOPATH=$HOME/go
export PATH=/usr/local/go/bin:$PATH:$GOPATH/bin
EOF
$ source ~/.bashrc
$ go version
go version go1.10.2 linux/arm
```

### Install dep — package manager for go and check it.

```bash
$ go get -u github.com/golang/dep/cmd/dep
$ sudo cp ~/go/bin/dep /usr/local/bin/
$ dep version
dep:
 version     : devel
 build date  :
 git hash    :
 go version  : go1.10.2
 go compiler : gc
 platform    : linux/arm
 features    : ImportDuringSolve=false
```

## Install Fn Server

### Download
```bash
$ git clone https://github.com/fnproject/fn.git ${GOPATH}/src/github.com/fnproject/fn
$ cd ${GOPATH}/src/github.com/fnproject/fn
```

### Build
Install dependencies, build from sources and run it:

```
$ cd ${GOPATH}/src/github.com/fnproject/fn
$ make dep
$ make build
$ sudo ./fnserver
```

Now we that server is running =)

![fnproject_on_raspi.png](/assets/fnproject_on_raspi.png)

### Test
```bash
$ curl localhost:8080
{“goto”:”https://github.com/fnproject/fn","hello":"world!"}
```

### Inatsll

```bash
$ sudo cp ~/go/src/github.com/fnproject/fn/fnserver /usr/local/bin/
$ sudo bash -c 'cat << EOF >> /etc/systemd/system/fnserver.service
[Unit]
Description=Fn Server
[Service]
Restart=always
ExecStart=/usr/local/bin/fnserver
[Install]
WantedBy=multi-user.target
EOF'
```

```bash
$ systemctl enable fnserver
Created symlink /etc/systemd/system/multi-user.target.wants/fnserver.service → /etc/systemd/system/fnserver.service.
$ systemctl start fnserver
$ status fnserver
● fnserver.service - Fn Server
   Loaded: loaded (/etc/systemd/system/fnserver.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-07-20 11:10:49 EDT; 9s ago
 Main PID: 18232 (fnserver)
   Memory: 3.6M
      CPU: 104ms
   CGroup: /system.slice/fnserver.service
           └─18232 /usr/local/bin/fnserver

Jul 20 11:10:49 raspberrypi-5 fnserver[18232]: time="2018-07-20T11:10:49-04:00" level=info msg="sync and async ram reservations" availMemory=495763456 ramAsync=396610765 ramAsyncHWMark=317288612 r
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]: time="2018-07-20T11:10:49-04:00" level=info msg="available cpu" availCPU=4000 totalCPU=4000
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]: time="2018-07-20T11:10:49-04:00" level=info msg="sync and async cpu reservations" cpuAsync=3200 cpuAsyncHWMark=2560 cpuSync=800
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]:         ______
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]:        / ____/___
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]:       / /_  / __ \
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]:      / __/ / / / /
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]:     /_/   /_/ /_/
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]:         v0.3.511
Jul 20 11:10:49 raspberrypi-5 fnserver[18232]: time="2018-07-20T11:10:49-04:00" level=info msg="Fn serving on `:8080`" type=full
```

## Install command line interface

```bash
$ git clone https://github.com/fnproject/cli.git ${GOPATH}/src/github.com/fnproject/cli
$ cd ${GOPATH}/src/github.com/fnproject/cli
$ make dep
$ make build
$ sudo cp ~/go/src/github.com/fnproject/cli/fn /usr/local/bin/
$ fn version
Client version is latest version: 0.4.131
Server version:  0.3.511
```

## Build Golang containers

```bash
$ docker pull arm32v6/golang:1.10.3-alpine
$ docker tag arm32v6/golang:1.10.3-alpine fnproject/go
$ cd /tmp
$ cat << EOF >> Dockerfile
FROM arm32v6/golang:1.10.3-alpine
RUN apk add --no-cache wget curl git bzr mercurial build-base
EOF
$ docker build -t fnproject/go:dev .
```

## First project

```bash
$ fn init --runtime go hello
Creating function at: /hello
Runtime: go
Function boilerplate generated.
func.yaml created.
$ cd hello/
$ export FN_REGISTRY=rajr
$
$ fn run
Building image rajr/hello:0.0.1 ............
{"message":"Hello World"}
```
```bash
$ fn -v  run
Building image rajr/hello:0.0.2
Sending build context to Docker daemon  13.31kB
Step 1/10 : FROM fnproject/go:dev as build-stage
 ---> a7de8cef7819
Step 2/10 : WORKDIR /function
 ---> Running in c59c86db7963
Removing intermediate container c59c86db7963
 ---> f819c87709e4
Step 3/10 : RUN go get -u github.com/golang/dep/cmd/dep
 ---> Running in 9479728f0b0c
Removing intermediate container 9479728f0b0c
 ---> 5f10dc0e6c2e
Step 4/10 : ADD . /go/src/func/
 ---> d816561d202b
Step 5/10 : RUN cd /go/src/func/ && dep ensure
 ---> Running in bb12992a5a71
Removing intermediate container bb12992a5a71
 ---> 6d0977aabba0
Step 6/10 : RUN cd /go/src/func/ && go build -o func
 ---> Running in 3250515d0d0b
Removing intermediate container 3250515d0d0b
 ---> 34dffb534b1c
Step 7/10 : FROM fnproject/go
 ---> 6d6cda97f54c
Step 8/10 : WORKDIR /function
 ---> Using cache
 ---> d218780f6880
Step 9/10 : COPY --from=build-stage /go/src/func/func /function/
 ---> a006806c5370
Step 10/10 : ENTRYPOINT ["./func"]
 ---> Running in c366e30f463d
Removing intermediate container c366e30f463d
 ---> c1736822399c
Successfully built c1736822399c
Successfully tagged rajr/hello:0.0.2

{"message":"Hello World"}
```

