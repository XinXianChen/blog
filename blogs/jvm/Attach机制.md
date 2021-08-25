# JVM Attach机制实现

## Attach是什么
那Attach机制是什么？说简单点就是jvm提供一种jvm进程间通信的能力，能让一个进程传命令给另外一个进程，并让它执行内部的一些操作，比如说我们为了让另外一个jvm进程把线程dump出来，那么我们跑了一个jstack的进程，然后传了个pid的参数，告诉它要哪个进程进行线程dump，既然是两个进程，那肯定涉及到进程间通信，以及传输协议的定义，比如要执行什么操作，传了什么参数等。

## Attach能做些什么
总结起来说，比如内存dump，线程dump，类信息统计(比如加载的类及大小以及实例个数等)，动态加载agent(使用过btrace的应该不陌生)，动态设置vm flag(但是并不是所有的flag都可以设置的，因为有些flag是在jvm启动过程中使用的，是一次性的)，打印vm flag，获取系统属性等，这些对应的源码(AttachListener.cpp)如下
```
static AttachOperationFunctionInfo funcs[] = {
  { "agentProperties",  get_agent_properties },
  { "datadump",         data_dump },
  { "dumpheap",         dump_heap },
  { "load",             JvmtiExport::load_agent_library },
  { "properties",       get_system_properties },
  { "threaddump",       thread_dump },
  { "inspectheap",      heap_inspection },
  { "setflag",          set_flag },
  { "printflag",        print_flag },
  { "jcmd",             jcmd },
  { NULL,               NULL }
};
```
## Attach在jvm里如何实现的

jvm在启动过程中可能并没有启动Attach Listener这个线程，可以通过jvm参数来启动，代码 （Threads::create_vm）如下：
```
if (!DisableAttachMechanism) {
    if (StartAttachListener || AttachListener::init_at_startup()) {
      AttachListener::init();
    }
  }
bool AttachListener::init_at_startup() {
  if (ReduceSignalUsage) {
    return true;
  } else {
    return false;
  }
}
```
其中DisableAttachMechanism，StartAttachListener ，ReduceSignalUsage均默认是false(globals.hpp)

```
product(bool, DisableAttachMechanism, false,                              \
         "Disable mechanism that allows tools to Attach to this VM”)   
product(bool, StartAttachListener, false,                                 \
          "Always start Attach Listener at VM startup")  
product(bool, ReduceSignalUsage, false,                                   \
          "Reduce the use of OS signals in Java and/or the VM”)  
```
  因此AttachListener::init()并不会被执行，而Attach Listener线程正是在此方法里创建的

```
// Starts the Attach Listener thread
void AttachListener::init() {
  EXCEPTION_MARK;
  klassOop k = SystemDictionary::resolve_or_fail(vmSymbols::java_lang_Thread(), true, CHECK);
  instanceKlassHandle klass (THREAD, k);
  instanceHandle thread_oop = klass->allocate_instance_handle(CHECK);

  const char thread_name[] = "Attach Listener";
  Handle string = java_lang_String::create_from_str(thread_name, CHECK);

  // Initialize thread_oop to put it into the system threadGroup
  Handle thread_group (THREAD, Universe::system_thread_group());
  JavaValue result(T_VOID);
  JavaCalls::call_special(&result, thread_oop,
                       klass,
                       vmSymbols::object_initializer_name(),
                       vmSymbols::threadgroup_string_void_signature(),
                       thread_group,
                       string,
                       CHECK);

  KlassHandle group(THREAD, SystemDictionary::ThreadGroup_klass());
  JavaCalls::call_special(&result,
                        thread_group,
                        group,
                        vmSymbols::add_method_name(),
                        vmSymbols::thread_void_signature(),
                        thread_oop,             // ARG 1
                        CHECK);

  { MutexLocker mu(Threads_lock);
    JavaThread* listener_thread = new JavaThread(&Attach_listener_thread_entry);

    // Check that thread and osthread were created
    if (listener_thread == NULL || listener_thread->osthread() == NULL) {
      vm_exit_during_initialization("java.lang.OutOfMemoryError",
                                    "unable to create new native thread");
    }

    java_lang_Thread::set_thread(thread_oop(), listener_thread);
    java_lang_Thread::set_daemon(thread_oop());

    listener_thread->set_threadObj(thread_oop());
    Threads::add(listener_thread);
    Thread::start(listener_thread);
  }
}
```

既然在启动的时候不会创建这个线程，那么我们在上面看到的那个线程是怎么创建的呢，这个就要关注另外一个线程“Signal Dispatcher”了，顾名思义是处理信号的，这个线程是在jvm启动的时候就会创建的。

下面以jstack的实现来说明触发Attach这一机制进行的过程，jstack命令的实现其实是一个叫做JStack.java的类，查看jstack代码后会走到下面的方法里

```
private static void runThreadDump(String pid, String args[]) throws Exception {
        VirtualMachine vm = null;
        try {
            vm = VirtualMachine.Attach(pid);
        } catch (Exception x) {
            String msg = x.getMessage();
            if (msg != null) {
                System.err.println(pid + ": " + msg);
            } else {
                x.printStackTrace();
            }
            if ((x instanceof AttachNotSupportedException) &&
                (loadSAClass() != null)) {
                System.err.println("The -F option can be used when the target " +
                    "process is not responding");
            }
            System.exit(1);
        }

        // Cast to HotSpotVirtualMachine as this is implementation specific
        // method.
        InputStream in = ((HotSpotVirtualMachine)vm).remoteDataDump((Object[])args);

        // read to EOF and just print output
        byte b[] = new byte[256];
        int n;
        do {
            n = in.read(b);
            if (n > 0) {
                String s = new String(b, 0, n, "UTF-8");
                System.out.print(s);
            }
        } while (n > 0);
        in.close();
        vm.detach();
    }
```
请注意VirtualMachine.Attach(pid);这行代码，触发Attach pid的关键，如果是在linux下会走到下面的构造函数。

```
LinuxVirtualMachine(AttachProvider provider, String vmid)
        throws AttachNotSupportedException, IOException
    {
        super(provider, vmid);

        // This provider only understands pids
        int pid;
        try {
            pid = Integer.parseInt(vmid);
        } catch (NumberFormatException x) {
            throw new AttachNotSupportedException("Invalid process identifier");
        }

        // Find the socket file. If not found then we attempt to start the
        // Attach mechanism in the target VM by sending it a QUIT signal.
        // Then we attempt to find the socket file again.
        path = findSocketFile(pid);
        if (path == null) {
            File f = createAttachFile(pid);
            try {
                // On LinuxThreads each thread is a process and we don't have the
                // pid of the VMThread which has SIGQUIT unblocked. To workaround
                // this we get the pid of the "manager thread" that is created
                // by the first call to pthread_create. This is parent of all
                // threads (except the initial thread).
                if (isLinuxThreads) {
                    int mpid;
                    try {
                        mpid = getLinuxThreadsManager(pid);
                    } catch (IOException x) {
                        throw new AttachNotSupportedException(x.getMessage());
                    }
                    assert(mpid >= 1);
                    sendQuitToChildrenOf(mpid);
                } else {
                    sendQuitTo(pid);
                }

                // give the target VM time to start the Attach mechanism
                int i = 0;
                long delay = 200;
                int retries = (int)(AttachTimeout() / delay);
                do {
                    try {
                        Thread.sleep(delay);
                    } catch (InterruptedException x) { }
                    path = findSocketFile(pid);
                    i++;
                } while (i <= retries && path == null);
                if (path == null) {
                    throw new AttachNotSupportedException(
                        "Unable to open socket file: target process not responding " +
                        "or HotSpot VM not loaded");
                }
            } finally {
                f.delete();
            }
        }

        // Check that the file owner/permission to avoid Attaching to
        // bogus process
        checkPermissions(path);

        // Check that we can connect to the process
        // - this ensures we throw the permission denied error now rather than
        // later when we attempt to enqueue a command.
        int s = socket();
        try {
            connect(s, path);
        } finally {
            close(s);
        }
    }

```
这里要解释下代码了，首先看到调用了createAttachFile方法在目标进程的cwd目录下创建了一个文件/proc//cwd/.Attach_pid，这个在后面的信号处理过程中会取出来做判断(为了安全)，另外我们知道在linux下线程是用进程实现的，在jvm启动过程中会创建很多线程，比如我们上面的信号线程，也就是会看到很多的pid(应该是LWP)，那么如何找到这个信号处理线程呢，从上面实现来看是找到我们传进去的pid的父进程，然后给它的所有子进程都发送一个SIGQUIT信号，而jvm里除了信号线程，其他线程都设置了对此信号的屏蔽，因此收不到该信号，于是该信号就传给了“Signal Dispatcher”，在传完之后作轮询等待看目标进程是否创建了某个文件，AttachTimeout默认超时时间是5000ms，可通过设置系统变量sun.tools.Attach.AttachTimeout来指定，下面是Signal Dispatcher线程的entry实现

```
static void signal_thread_entry(JavaThread* thread, TRAPS) {
  os::set_priority(thread, NearMaxPriority);
  while (true) {
    int sig;
    {
      // FIXME : Currently we have not decieded what should be the status
      //         for this java thread blocked here. Once we decide about
      //         that we should fix this.
      sig = os::signal_wait();
    }
    if (sig == os::sigexitnum_pd()) {
       // Terminate the signal thread
       return;
    }

    switch (sig) {
      case SIGBREAK: {
        // Check if the signal is a trigger to start the Attach Listener - in that
        // case don't print stack traces.
        if (!DisableAttachMechanism && AttachListener::is_init_trigger()) {
          continue;
        }
        // Print stack traces
        // Any SIGBREAK operations added here should make sure to flush
        // the output stream (e.g. tty->flush()) after output.  See 4803766.
        // Each module also prints an extra carriage return after its output.
        VM_PrintThreads op;
        VMThread::execute(&op);
        VM_PrintJNI jni_op;
        VMThread::execute(&jni_op);
        VM_FindDeadlocks op1(tty);
        VMThread::execute(&op1);
        Universe::print_heap_at_SIGBREAK();
        if (PrintClassHistogram) {
          VM_GC_HeapInspection op1(gclog_or_tty, true /* force full GC before heap inspection */,
                                   true /* need_prologue */);
          VMThread::execute(&op1);
        }
        if (JvmtiExport::should_post_data_dump()) {
          JvmtiExport::post_data_dump();
        }
        break;
      }
      ….
      }
    }
  }
}
```
当信号是SIGBREAK(在jvm里做了#define，其实就是SIGQUIT)的时候，就会触发 AttachListener::is_init_trigger()的执行

```
bool AttachListener::is_init_trigger() {
  if (init_at_startup() || is_initialized()) {
    return false;               // initialized at startup or already initialized
  }
  char fn[PATH_MAX+1];
  sprintf(fn, ".Attach_pid%d", os::current_process_id());
  int ret;
  struct stat64 st;
  RESTARTABLE(::stat64(fn, &st), ret);
  if (ret == -1) {
    snprintf(fn, sizeof(fn), "%s/.Attach_pid%d",
             os::get_temp_directory(), os::current_process_id());
    RESTARTABLE(::stat64(fn, &st), ret);
  }
  if (ret == 0) {
    // simple check to avoid starting the Attach mechanism when
    // a bogus user creates the file
    if (st.st_uid == geteuid()) {
      init();
      return true;
    }
  }
  return false;
}
```
一开始会判断当前进程目录下是否有个.Attach_pid文件（前面提到了），如果没有就会在/tmp下创建一个/tmp/.Attach_pid，当那个文件的uid和自己的uid是一致的情况下（为了安全）再调用init方法。
```
// Starts the Attach Listener thread
void AttachListener::init() {
  EXCEPTION_MARK;
  klassOop k = SystemDictionary::resolve_or_fail(vmSymbols::java_lang_Thread(), true, CHECK);
  instanceKlassHandle klass (THREAD, k);
  instanceHandle thread_oop = klass->allocate_instance_handle(CHECK);

  const char thread_name[] = "Attach Listener";
  Handle string = java_lang_String::create_from_str(thread_name, CHECK);

  // Initialize thread_oop to put it into the system threadGroup
  Handle thread_group (THREAD, Universe::system_thread_group());
  JavaValue result(T_VOID);
  JavaCalls::call_special(&result, thread_oop,
                       klass,
                       vmSymbols::object_initializer_name(),
                       vmSymbols::threadgroup_string_void_signature(),
                       thread_group,
                       string,
                       CHECK);

  KlassHandle group(THREAD, SystemDictionary::ThreadGroup_klass());
  JavaCalls::call_special(&result,
                        thread_group,
                        group,
                        vmSymbols::add_method_name(),
                        vmSymbols::thread_void_signature(),
                        thread_oop,             // ARG 1
                        CHECK);

  { MutexLocker mu(Threads_lock);
    JavaThread* listener_thread = new JavaThread(&Attach_listener_thread_entry);

    // Check that thread and osthread were created
    if (listener_thread == NULL || listener_thread->osthread() == NULL) {
      vm_exit_during_initialization("java.lang.OutOfMemoryError",
                                    "unable to create new native thread");
    }

    java_lang_Thread::set_thread(thread_oop(), listener_thread);
    java_lang_Thread::set_daemon(thread_oop());

    listener_thread->set_threadObj(thread_oop());
    Threads::add(listener_thread);
    Thread::start(listener_thread);
  }
}
```
  此时水落石出了，看到创建了一个线程，并且取名为Attach Listener。再看看其子类LinuxAttachListener的init方法

```
int LinuxAttachListener::init() {
  char path[UNIX_PATH_MAX];          // socket file
  char initial_path[UNIX_PATH_MAX];  // socket file during setup
  int listener;                      // listener socket (file descriptor)

  // register function to cleanup
  ::atexit(listener_cleanup);

  int n = snprintf(path, UNIX_PATH_MAX, "%s/.java_pid%d",
                   os::get_temp_directory(), os::current_process_id());
  if (n < (int)UNIX_PATH_MAX) {
    n = snprintf(initial_path, UNIX_PATH_MAX, "%s.tmp", path);
  }
  if (n >= (int)UNIX_PATH_MAX) {
    return -1;
  }

  // create the listener socket
  listener = ::socket(PF_UNIX, SOCK_STREAM, 0);
  if (listener == -1) {
    return -1;
  }

  // bind socket
  struct sockaddr_un addr;
  addr.sun_family = AF_UNIX;
  strcpy(addr.sun_path, initial_path);
  ::unlink(initial_path);
  int res = ::bind(listener, (struct sockaddr*)&addr, sizeof(addr));
  if (res == -1) {
    RESTARTABLE(::close(listener), res);
    return -1;
  }

  // put in listen mode, set permissions, and rename into place
  res = ::listen(listener, 5);
  if (res == 0) {
      RESTARTABLE(::chmod(initial_path, S_IREAD|S_IWRITE), res);
      if (res == 0) {
          res = ::rename(initial_path, path);
      }
  }
  if (res == -1) {
    RESTARTABLE(::close(listener), res);
    ::unlink(initial_path);
    return -1;
  }
  set_path(path);
  set_listener(listener);

  return 0;
}
```
看到其创建了一个监听套接字，并创建了一个文件/tmp/.java_pid，这个文件就是客户端之前一直在轮询等待的文件，随着这个文件的生成，意味着Attach的过程圆满结束了。

### Attach listener接收请求
  看看它的entry实现Attach_listener_thread_entry

```
static void Attach_listener_thread_entry(JavaThread* thread, TRAPS) {
  os::set_priority(thread, NearMaxPriority);

  thread->record_stack_base_and_size();

  if (AttachListener::pd_init() != 0) {
    return;
  }
  AttachListener::set_initialized();

  for (;;) {
    AttachOperation* op = AttachListener::dequeue();
    if (op == NULL) {
      return;   // dequeue failed or shutdown
    }

    ResourceMark rm;
    bufferedStream st;
    jint res = JNI_OK;

    // handle special detachall operation
    if (strcmp(op->name(), AttachOperation::detachall_operation_name()) == 0) {
      AttachListener::detachall();
    } else {
      // find the function to dispatch too
      AttachOperationFunctionInfo* info = NULL;
      for (int i=0; funcs[i].name != NULL; i++) {
        const char* name = funcs[i].name;
        assert(strlen(name) <= AttachOperation::name_length_max, "operation <= name_length_max");
        if (strcmp(op->name(), name) == 0) {
          info = &(funcs[i]);
          break;
        }
      }

      // check for platform dependent Attach operation
      if (info == NULL) {
        info = AttachListener::pd_find_operation(op->name());
      }

      if (info != NULL) {
        // dispatch to the function that implements this operation
        res = (info->func)(op, &st);
      } else {
        st.print("Operation %s not recognized!", op->name());
        res = JNI_ERR;
      }
    }

    // operation complete - send result and output to client
    op->complete(res, &st);
  }
}

```
代码来看就是从队列里不断取AttachOperation，然后找到请求命令对应的方法进行执行，比如我们一开始说的jstack命令，找到 { “threaddump”, thread_dump }的映射关系，然后执行thread_dump方法。

  再来看看其要调用的AttachListener::dequeue()，

```
AttachOperation* AttachListener::dequeue() {
  JavaThread* thread = JavaThread::current();
  ThreadBlockInVM tbivm(thread);

  thread->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition() or
  // java_suspend_self() via check_and_wait_while_suspended()

  AttachOperation* op = LinuxAttachListener::dequeue();

  // were we externally suspended while we were waiting?
  thread->check_and_wait_while_suspended();

  return op;
}

```
  最终调用的是LinuxAttachListener::dequeue()，

```
LinuxAttachOperation* LinuxAttachListener::dequeue() {
  for (;;) {
    int s;

    // wait for client to connect
    struct sockaddr addr;
    socklen_t len = sizeof(addr);
    RESTARTABLE(::accept(listener(), &addr, &len), s);
    if (s == -1) {
      return NULL;      // log a warning?
    }

    // get the credentials of the peer and check the effective uid/guid
    // - check with jeff on this.
    struct ucred cred_info;
    socklen_t optlen = sizeof(cred_info);
    if (::getsockopt(s, SOL_SOCKET, SO_PEERCRED, (void*)&cred_info, &optlen) == -1) {
      int res;
      RESTARTABLE(::close(s), res);
      continue;
    }
    uid_t euid = geteuid();
    gid_t egid = getegid();

    if (cred_info.uid != euid || cred_info.gid != egid) {
      int res;
      RESTARTABLE(::close(s), res);
      continue;
    }

    // peer credential look okay so we read the request
    LinuxAttachOperation* op = read_request(s);
    if (op == NULL) {
      int res;
      RESTARTABLE(::close(s), res);
      continue;
    } else {
      return op;
    }
  }
}

```
我们看到如果没有请求的话，会一直accept在那里，当来了请求，然后就会创建一个套接字，并读取数据，构建出LinuxAttachOperation返回并执行。

  整个过程就这样了，从Attach线程创建到接收请求，处理请求。


**转自：http://lovestblog.cn/blog/2014/06/18/jvm-attach/**


