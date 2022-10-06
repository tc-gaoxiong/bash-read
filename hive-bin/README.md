# Hive bin 目录下 bash 脚本阅读

多次翻阅这些脚本之后，我决定彻底搞懂它，并记录下来。

## 启动 hiveserver2，即执行 `./bin/hiveserver2` 后发生了什么

hiveserver2 脚本只有一个作用，就是调用 hive 脚本：

```bash
. "$bin"/hive --service hiveserver2 "$@"
```

`$@` 的作用是传递参数，hiveserver2 只有两个选项，其中一个是帮助命令，另一个是 `--hiveconf <property=value>`，用于传递配置参数。

hive 脚本会接收到第一个参数是指定启动的服务为 hiveserver2，如下是 hive 脚本处理参数的部分逻辑：

```bash
SERVICE="" # 变量声明

# ...

SERVICE_ARGS=() # 声明一个数组
while [ $# -gt 0 ]; do # $# - 参数数量，这里为 2 --service hiveserver2
  case "$1" in
    
    # ...

    --service) # 匹配 --vervice
      shift # 默认左移一位
      SERVICE=$1 # 左移之后 $1 值为 hiveserver2
      shift # 再左移，处理剩余的参数
      ;;
   
    # ...
    
    *)
      SERVICE_ARGS=("${SERVICE_ARGS[@]}" "$1")
      # 数组元素的获取方式为：${name[subscript]}
      # ${SERVICE_ARGS[@]} - 下标为 @ 或 * 时表示读取数组内全部元素
      # 此时 $1 为传递给 hiveserver2 的参数，会依次遍历放入到数组中
      shift
      ;;
  esac
done
```

接下来是各种判断、加载配置等，直接到 339 行：

```bash
SERVICE_LIST="" # 变量声明

# 接下来的两个 for 循环将加载 ext 目录下的所有脚本，其中就有一个 hiveserver2 的脚本
for i in "$bin"/ext/*.sh ; do
  . $i
done

for i in "$bin"/ext/util/*.sh ; do
  . $i
done
```

接着阅读 ext/hiveserver2.sh 脚本：

```bash
THISSERVICE=hiveserver2
export SERVICE_LIST="${SERVICE_LIST}${THISSERVICE} "
```

前两行代码是拼接 server 列表，用空格分隔。然后都是函数定义，暂时先不看。返回到 hive 脚本中，跳到 359 行：

```bash
TORUN="" # 变量声明
for j in $SERVICE_LIST ; do # 遍历 
  if [ "$j" = "$SERVICE" ] ; then
    TORUN=${j}$HELP # 这里 TORUN=hiveserver2
  fi
done

# ...

if [[ "$SERVICE" =~ ^(hiveserver2|beeline|cli)$ ]] ; then
    # [[]] 表示在等量运算符右侧启用 shell 风格的模式匹配
                 # =~ 正则匹配，这里匹配到 hiveserver2
  # If process is backgrounded, don't change terminal settings
  if [[ ( ! $(ps -o stat= -p $$) =~ "+" ) && ! ( -p /dev/stdin ) && ( ! $(ps -o tty= -p $$) =~ "?" ) ]]; then
    export HADOOP_CLIENT_OPTS="$HADOOP_CLIENT_OPTS -Djline.terminal=jline.UnsupportedTerminal"
            # -o stat= 指定输出格式，只输出 stat 列
            # STAT  进程状态，用下面的代码中的一个给出： 
            #     D    不可中断     Uninterruptible sleep (usually IO)
            #     R    正在运行，或在队列中的进程
            #     S    处于休眠状态
            #     T    停止或被追踪
            #     Z    僵尸进程
            #     W    进入内存交换（从内核2.6开始无效）
            #     X    死掉的进程
            #     <    高优先级
            #     N    低优先级
            #     L    有些页被锁进内存
            #     s    包含子进程
            #     +    位于后台的进程组；
            #     l    多线程，克隆线程
            # TTY   进程相关的终端，? 表示没终端
            # $$ 当前进程的进程号 pid

            # -p /dev/stdin TODO : 不懂
  fi
fi

export JVM_PID="$$"

if [ "$TORUN" = "" ] ; then
  echo "Service $SERVICE not found"
  echo "Available Services: $SERVICE_LIST"
  exit 7
else
  set -- "${SERVICE_ARGS[@]}"
  # set 可以改变 shell 位置的值并且设置位置参数
  # set -- -a -b -c 等等，就是重置位置参数为 -a -b -c
  $TORUN "$@" # 实际是调用 ext/hiveserver2.sh 中的 hiveserver2() 函数
fi
```

hiveserver2() 函数如下：

```bash
hiveserver2() {
  CLASS=org.apache.hive.service.server.HiveServer2
  if $cygwin; then
    HIVE_LIB=`cygpath -w "$HIVE_LIB"`v
  fi
  JAR=${HIVE_LIB}/hive-service-[0-9].*.jar

  if [ "$1" = "--graceful_stop" ]; then # 停止 hiveserver2，这里跳过
    pid=$2
    if [ "$pid" = "" -a -f $HIVESERVER2_PID ]; then
      pid=$(cat $HIVESERVER2_PID)
    fi
    TIMEOUT_KEY='hive.server2.graceful.stop.timeout'
    timeout=$(exec $HADOOP jar $JAR $CLASS $HIVE_OPTS --getHiveConf $TIMEOUT_KEY | grep $TIMEOUT_KEY'=' | awk -F'=' '{print $2}')
    killAndWait $pid $timeout
  else
    export HADOOP_CLIENT_OPTS=" -Dproc_hiveserver2 $HADOOP_CLIENT_OPTS "
    export HADOOP_OPTS="$HIVESERVER2_HADOOP_OPTS $HADOOP_OPTS"
    commands=$(exec $HADOOP jar $JAR $CLASS -H | grep -v '-hiveconf' | awk '{print $1}')
            # $(command) 等同于 `command`
            # $HADOOP 在 hive 脚本中有定义：HADOOP=$HADOOP_HOME/bin/hadoop
            # TODO : 涉及到 hadoop 脚本，搁置，不过这个函数到这儿就很明确了，通过 hadoop 启动 hiveserver2
    start_hiveserver2='Y'
    for i in "$@"; do
      if [ $(echo "${commands[@]}" | grep -we "$i") != "" ]; then
        start_hiveserver2='N'
        break
      fi
    done
    if [ "$start_hiveserver2" == "Y" ]; then
      before_start
      echo >&2 "$(timestamp): Starting HiveServer2"
      hiveserver2_pid="$$"
      echo $hiveserver2_pid > ${HIVESERVER2_PID}
    fi
    exec $HADOOP jar $JAR $CLASS $HIVE_OPTS "$@"
  fi
}
```