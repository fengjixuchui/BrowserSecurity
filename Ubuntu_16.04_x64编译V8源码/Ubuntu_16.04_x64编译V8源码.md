# Ubuntu 16.04 x64编译V8源码

**Author:wnagzihxa1n
E-Mail:wnagzihxa1n@gmail.com**

之前编译Chromium遇到了各种问题，先是Ubuntu下编译，最后东拼西凑跑起来了，然后在Windows 10下编译，也跑起来了，不过Debug版本下断点啥的也有问题，Release版本的直接编译不了，在跟一位师傅聊了聊之后我决定先不弄Chromium这么大的工程了，转为搞V8，虽然也很大，但是起码相比Chromium小那么些许

前面的步骤都是一样的，先安装`depot_tools`，但是要先配置翻墙环境，默认本地代理
```
127.0.0.1:1080
```

设置git代理
```
potat0@v-ubuntu:~$ git config --global http.proxy socks5://127.0.0.1:1080
potat0@v-ubuntu:~$ git config --global https.proxy socks5://127.0.0.1:1080
```

安装`polipo`设置终端代理
```
potat0@v-ubuntu:~$ sudo apt-get install polipo
```

修改配置
```
potat0@v-ubuntu:~$ sudo gedit /etc/polipo/config
```

写入
```
# This file only needs to list configuration variables that deviate
# from the default values.  See /usr/share/doc/polipo/examples/config.sample
# and "polipo -v" for variables you can tweak and further information.

logSyslog = false
logFile = "/var/log/polipo/polipo.log"

socksParentProxy = "127.0.0.1:1080"
socksProxyType = socks5

chunkHighMark = 50331648
objectHighMark = 16384

serverMaxSlots = 64
serverSlots = 16
serverSlots1 = 32

proxyAddress = "0.0.0.0"
proxyPort = 8123
```

重启`polipo`生效并设置环境变量
```
potat0@v-ubuntu:~$ /etc/init.d/polipo restart
potat0@v-ubuntu:~/v8/v8$ export https_proxy="http://localhost:8123/"
potat0@v-ubuntu:~/v8/v8$ export https_proxy="http://localhost:8123/"
```

创建`.boto`文件
```
[Boto]
proxy=127.0.0.1
proxy_port=8123
```

导出环境变量
```
set NO_AUTH_BOTO_CONFIG=/home/potat0/.boto
```

开始下载工具到本地
```
potat0@v-ubuntu:~$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

向`.bashrc`中写入上述工具的路径，并source生效

创建V8目录，并进入获取V8源码
```
potat0@v-ubuntu:~$ mkdir v8
potat0@v-ubuntu:~$ cd v8
potat0@v-ubuntu:~/v8$ fetch v8
```

中途断开使用
```
potat0@v-ubuntu:~/v8$ gclient sync
```

正常情况下可以完整下完V8的源码，不过还是会有错误的

如果遇到下面这个错误，说明是`cipd`联网出问题了，因为curl没法通过我们的代理下载，所以我们需要手动修改一下
```
potat0@v-ubuntu:~/v8$ gclient sync
curl: (7) Failed to connect to chrome-infra-packages.appspot.com port 443: Connection refused
Syncing projects: 100% (22/22), done.                                               

________ running 'cipd ensure -log-level error -root /home/potat0/v8 -ensure-file /tmp/tmpwH7Lag.ensure' in '.'
curl: (7) Failed to connect to chrome-infra-packages.appspot.com port 443: Connection refused
Error: Command 'cipd ensure -log-level error -root /home/potat0/v8 -ensure-file /tmp/tmpwH7Lag.ensure' returned non-zero exit status 7
```

首先根据打印的字符串搜索关键的代码在`cipd`这个文件，我们通过将构造的地址输出，手动下载该文件即可
```
# CIPD_BACKEND can be changed to ...-dev for manual testing.
CIPD_BACKEND="https://chrome-infra-packages.appspot.com"
VERSION_FILE="${MYPATH}/cipd_client_version"

CLIENT="${MYPATH}/.cipd_client"
VERSION=`cat "${VERSION_FILE}"`
PLATFORM="${OS}-${ARCH}"

URL="${CIPD_BACKEND}/client?platform=${PLATFORM}&version=${VERSION}"
echo "${URL}" <===
USER_AGENT="depot_tools/$(git -C ${MYPATH} rev-parse HEAD 2>/dev/null || echo "???")"
```

重新运行，可以看到已经输出了关键的地址
```
potat0@v-ubuntu:~/v8$ gclient sync
depot_tools update failed. Conflict in /home/potat0/depot_tools
Cannot rebase: You have unstaged changes.
Please commit or stash them.
https://chrome-infra-packages.appspot.com/client?platform=linux-amd64&version=git_revision:fb963f0f43e265a65fb7f1f202e17ea23e947063
curl: (51) SSL: certificate subject name (*.atlassolutions.com) does not match target host name 'chrome-infra-packages.appspot.com'
Syncing projects: 100% (22/22), done.                                               

________ running 'cipd ensure -log-level error -root /home/potat0/v8 -ensure-file /tmp/tmpDryQBh.ensure' in '.'
https://chrome-infra-packages.appspot.com/client?platform=linux-amd64&version=git_revision:fb963f0f43e265a65fb7f1f202e17ea23e947063
curl: (7) Failed to connect to chrome-infra-packages.appspot.com port 443: Connection refused
Error: Command 'cipd ensure -log-level error -root /home/potat0/v8 -ensure-file /tmp/tmpDryQBh.ensure' returned non-zero exit status 7
```

我们通过手动下载放到根目录就行

按照上述的方法，如果下载完V8源码后会碰到下面这个错误
```
potat0@v-ubuntu:~/v8/v8$ gclient sync
Syncing projects: 100% (22/22), done.                                               

________ running 'download_from_google_storage --no_resume --platform=linux* --no_auth --bucket chromium-clang-format -s v8/buildtools/linux64/clang-format.sha1' in '/home/potat0/v8'

Error: Command 'download_from_google_storage --no_resume --platform=linux* --no_auth --bucket chromium-clang-format -s v8/buildtools/linux64/clang-format.sha1' returned non-zero exit status 1 in /home/potat0/v8
Hook 'download_from_google_storage --no_resume '--platform=linux*' --no_auth --bucket chromium-clang-format -s v8/buildtools/linux64/clang-format.sha1' took 33.30 secs
```

可以手动下载，我举个例子，比如上面要下载的是`clang-format`，我们就按照下面这个格式拼接，其中`https://storage.googleapis.com`是固定的头部，中间的`chromium-clang-format`是报错信息`--bucket`后面跟着的参数，最后的`5349d1954e17f6ccafb6e6663b0f13cdb2bb33c8`是文本文件`v8/buildtools/linux64/clang-format.sha1`的数据，所以只要碰到这种错误就手动下载，然后放到`v8/buildtools/linux64/clang-format.sha1`同目录下即可，记住要重命名，比如`clang-format.sha1`这个文件对应的就是`clang-format`
```
https://storage.googleapis.com/chromium-clang-format/5349d1954e17f6ccafb6e6663b0f13cdb2bb33c8
```

若干例子，比如下面第三个，是一个`tar.gz`压缩文件，需要下载后手动解压缩出来，解压缩同目录即可
```
https://storage.googleapis.com/chromium-clang-format/5349d1954e17f6ccafb6e6663b0f13cdb2bb33c8
https://storage.googleapis.com/chromium-gn/1756964fe6b9f0a865accdf577ae46345847de3b
https://storage.googleapis.com/v8-wasm-spec-tests/90f3b976c7e98ee0e9539975189959dfd78b3480
```

这其中会有一个很神奇的操作，在下载``的时候，会莫名其妙卡住
```
potat0@v-ubuntu:~/v8/v8$ gclient sync
Syncing projects: 100% (22/22), done.                                               
^CFailed while running "/usr/bin/python v8/third_party/binutils/download.py"
Hook '/usr/bin/python v8/third_party/binutils/download.py' took 279.90 secs
Interrupted
```

于是我开了一个新的终端，没有设置其它配置，秒秒钟下载好
```
potat0@v-ubuntu:~/v8/v8$ gclient sync
Syncing projects: 100% (22/22), done.                                               

________ running '/usr/bin/python v8/third_party/binutils/download.py' in '/home/potat0/v8'
0> Downloading /home/potat0/v8/v8/third_party/binutils/Linux_x64/binutils.tar.bz2...
Downloading 1 files took 8.540997 second(s)
```

下载完后，开始下载依赖
```
potat0@v-ubuntu:~/v8/v8$ ./build/install-build-deps.sh --no-chromeos-fonts
```

万事俱备，开始生成编译文件，遇到了权限的错误
```
potat0@v-ubuntu:~/v8/v8$ sudo tools/dev/v8gen.py x64.debug -vv
[sudo] password for potat0: 
################################################################################
/usr/bin/python -u tools/mb/mb.py gen -f infra/mb/mb_config.pyl -m developer_default -b x64.debug out.gn/x64.debug
  
  Writing """\
  is_debug = true
  target_cpu = "x64"
  v8_enable_backtrace = true
  v8_enable_slow_dchecks = true
  v8_optimized_debug = false
  """ to /home/potat0/v8/v8/out.gn/x64.debug/args.gn.
  
  /home/potat0/v8/v8/buildtools/linux64/gn gen out.gn/x64.debug --check
  Traceback (most recent call last):
    File "tools/mb/mb.py", line 64, in Main
      ret = self.args.func()
    File "tools/mb/mb.py", line 287, in CmdGen
      return self.RunGNGen(vals)
    File "tools/mb/mb.py", line 698, in RunGNGen
      ret, _, _ = self.Run(cmd)
    File "tools/mb/mb.py", line 1117, in Run
      ret, out, err = self.Call(cmd, env=env, buffer_output=buffer_output)
    File "tools/mb/mb.py", line 1131, in Call
      env=env)
    File "/usr/lib/python2.7/subprocess.py", line 711, in __init__
      errread, errwrite)
    File "/usr/lib/python2.7/subprocess.py", line 1343, in _execute_child
      raise child_exception
  OSError: [Errno 13] Permission denied
Traceback (most recent call last):
  File "tools/dev/v8gen.py", line 304, in <module>
    sys.exit(gen.main())
  File "tools/dev/v8gen.py", line 298, in main
    return self._options.func()
  File "tools/dev/v8gen.py", line 166, in cmd_gen
    gn_outdir,
  File "tools/dev/v8gen.py", line 208, in _call_cmd
    stderr=subprocess.STDOUT,
  File "/usr/lib/python2.7/subprocess.py", line 574, in check_output
    raise CalledProcessError(retcode, cmd, output=output)
subprocess.CalledProcessError: Command '['/usr/bin/python', '-u', 'tools/mb/mb.py', 'gen', '-f', 'infra/mb/mb_config.pyl', '-m', 'developer_default', '-b', 'x64.debug', 'out.gn/x64.debug']' returned non-zero exit status 1
```

这个问题就是我们下载回来的文件，其实是没有执行权限的，所以我们给下载的文件执行权限即可
```
potat0@v-ubuntu:~/v8/v8$ sudo chmod 777 /home/potat0/v8/v8/buildtools/linux64/gn
potat0@v-ubuntu:~/v8/v8$ sudo chmod 777 /home/potat0/v8/v8/buildtools/linux64/clang-format
```

生成编译文件
```
potat0@v-ubuntu:~/v8/v8$ sudo tools/dev/v8gen.py x64.debug 
```

进行编译，错误真是一个接一个
```
out.gn/x64.debug'
ninja: error: toolchain.ninja:47: loading 'obj/bytecode_builtins_list_generator.ninja': Permission denied
subninja obj/bytecode_builtins_list_generator.ninja
                                                   ^ near here
```

有了上一个错误的教训之后，这一次我通过递归把全部文件设为可执行权限
```
potat0@v-ubuntu:~/v8$ sudo chmod 777 -R v8
```

再次编译就稳稳的了，最后生成的是一个`d8`的可执行文件











































