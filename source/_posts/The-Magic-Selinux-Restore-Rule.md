---
title: 神奇的Selinux Restore Rule
date: 2018-05-02 11:14:54
tags:
---

前些日子，老大的手机没电了，充电开机后一直卡在开机动画不断转啊转。立马将手机借来分析，由此便有了这篇文章。网上讲sepolicy的文章很多，尤其”[Innost大神的专栏](https://blog.csdn.net/innost/article/details/19299937) “讲的特别好（分一二两篇），所以其基础部分在此便不再赘述。

本文主要介绍的是Android Sepolicy对某些特定资料夹（如/data/data等）的独特管理方式，并介绍restorecon工具在Android系统中的使用注意事项。

## 案例：一个由selinux权限变化所引发的血案

前面说到老板的手机突然挂了，立马把手机借来，经过一系列操作后终于成功抓取到系统开机log，有了log很容易就定位到导致系统无法开机的直接原因：在Android Runtime做dexopt时，因无权限访问/data/app文件路径下的3rd party apk，从而造成java虚拟机重启/加载/crash的不断循环。 

对比正常机器，发现一个**神奇**的现象：**该机器中“/data/app”目录的Selinux权限居然被人改成了unlabeled**，如此一来java runtime的访问权限就活生生地被selinux挡死了。

通常况下的/data/app资料夹权限应为：
```
u:object_r:apk_data_file:s0       /data/app
```
而老大的手机中却变为了：
```
u:object_r:unlabeled:s0         /data/app
```

由此就引申出如下几个需要我们搞清楚的问题点：

- check 源代码，发现在init.rc的 post-fs-data默认就有做“restorecon /data”的行为，但为何开机后”/data/app“目录却没有被重新relabel？(在file_contexts 文件中已经定义过 /data/app(/.*)?   u:object_r:apk_data_file:s0)

- unlabeled的selinux label是如何被误打到/data/app文件夹中去的？

让我们来顺着这些问题，谈谈selinux针对某些特殊文件夹的label rule。


## Android系统中嵌入的selinux辅助工具

>  - getenforce : 获取当前selinux的开关状态。Enforcing : Selinux On; Permissive : Selinux OFF;

>  - setenforce :设置以开关selinux。 0 : 关闭 ;  1 : 打开;

> - chcon : 随意修改某个文件(夹)的selinux lable。Ex: chcon u:object_r:system_data_file:s0  /data/app

> - restorecon : 依照sepolicy Rule中定义的规则，重新relable指定的文件(夹)。

这里我主要要介绍下最后一条命令“restorecon”，其工作方式对解决前面提出的疑问有极大帮助。

```
usage: restorecon [-D] [-F] [-R] [-n] [-v] FILE...

Restores the default security contexts for the given files.

-D	apply to /data/data too
-F	force reset
-R	recurse into directories
-n	don't make any changes; useful with -v to see what would change
-v	verbose: show any changes
```

### restorecon -R（recurse）

默认不加参数状况下的restorecon命令与chcon命令的唯一差别的，前者会依照编译进系统的file_contexts rule来对指定的文件(夹)进行lable(在Android O之前，该rule文件放在boot img中)。

当我们加入-R参数后，该命令将会遍历所指定的文件夹，并依照编译进系统的file_contexts rule对每个文件进行label动作。注意：需要在repolicy enable的前提下才能正常使用。

值得注意的是，selinux的有如下规则：在未定义selinux label rule的前提下，文件夹中建立的新文件默认将会继承母文件夹的selinux label。当然，赋予特殊权限的进程是有能力修改新建文件selinux label的。

大家知道在Android系统中，app的数据资料默认会放到/data/data目录下以apk包名命名的各自资料夹中，根据apk享有的权限不同，其资料夹下的selinux label就自然会不一样，由于我们无法预先得知其需要的label，且file_contexts文件中定义的rule几乎都是通配符规则，故若不加以合理的管控，系统若调用了restorecon -R则/data/data下的selinux lable就全被改坏了。

针对此情况，Android系统对一些目录做了特殊处理，及时调用restorecon -R命令，依然直接skip relable过程，以保留Android 虚拟机对app selinux权限的管理能力。

例如：
```
ASUS_Z01R_1:/ # restorecon  -R /data/
SELinux: Loaded file_contexts
SELinux: Skipping restorecon_recursive(/data)
```
注意：如下这些目录都是-R命令会自动忽略处理label动作的：

```
/data/data
/data/user
/data/user_de
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user_de
```

### restorecon -D

在Android系统的实际实现中，该参数实际执行过程中，除了如help文档所描述的/data/data/也强行relabel外，对前面描述的
```
/data/data
/data/user
/data/user_de
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user_de
```
也同样会进行relabel动作。

实际实现文件为：
external/selinux/libselinux/src/android/android.c 
external/toybox/toys/android/restorecon.c 

### restorecon -F

前面有讲，在-R参数(recurse)加入后，虽然是遍历执行relabel模式，但有些文件夹
却被强行跳过relabel过程，说明有些情况系统已经打入的label其实并不需要频繁更新。

该参数的效果是，对除
> 1.  /data/data
/data/user
/data/user_de
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user_de
> 
> 2. /sys
> 
> 3. 文件node属于in-memory filesystems的RAMFS
> 
> 下文中我们把这些情况成为[情况A]

三种文件路径外的所有文件(夹)，强制进行relabel动作，一般需要搭配-R命令使用。

可能有人会有疑惑，前面的-R参数不是说对除某些特定文件夹外的所有文件进行relabel动作，为何这里还要进行force reset呢？

其实，为了防止在调用restorecon -R命令时，不必要的子文件夹selinux权限被无意义或错误地改动，所以对[情况A]外的所有节点，会多进行一次传入文件根路径的selinux label check，若该节点的selinux lable和编译预设的file_contexts规则一致，则直接跳出后续流程，即不进行任何子目录的遍历relabel过程。

加入-F参数后，selinux core就不再进行传入文件节点和预编译file_contexts规则的匹配判断，从而继续执行后续的relabel动作。不过
```
/data/data
/data/user
/data/user_de
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user
/mnt/expand/\?\?\?\?\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?-\?\?\?\?\?\?\?\?\?\?\?\?/user_de）
```
几个目录依然会自动跳出，不进行遍历以保证Android app的上层权限管理机制不被影响到。

## 深入理解Android init.rc 命令中集成的restorecon命令

前面我们学会了如何使用系统工具中带入的restorecon命令，其实init.rc中也集成了restorecon命令，不过并没有集成所有的功能(例如无-F功能支持)。

在init.rc中支援如下两个restorecon命令:
> restorecon
> restorecon_recursive

### rc文件中的restorecon命令

该命令在rc文件中对应前面讲过的工具中restorecon命令，不够所支援的参数为如下三个：
> --recursive： 遍历执行，同工具命令的restorecon -R
> --skip-ce ： 跳过”/data/system_ce/“和"/data/misc_ce/"目录的relabel遍历过程。
> --cross-filesystems ： open文件节点时，以少一个 FTS_XDEV flag的方式打开。由于用的比较少，故和不加该参数的差异还未做研究。

如rc文件中的实例：
```
restorecon --recursive --skip-ce /data
restorecon /adb_keys
```

###  rc文件中的restorecon_recursive命令

该命令其实就是对restorecon --recursive命令的封装，两者完全等价。
另外，该命令还是可以继续加入其他剩余的两个参数。



如rc文件中的实例：
```
 restorecon_recursive /mnt
 restorecon --recursive --cross-filesystems /sys/kernel/debug
 restorecon --recursive --skip-ce /data
```

### restorecon和restorecon_recursive在init中的具体实现

下面我们跟踪代码，探索Android系统中label过程的具体实现过程。

```
/* File name :/system/core/init/builtins.cpp*/
 static const Map builtin_functions = {
 ...
    {"restorecon",              {1,     kMax, do_restorecon}},
    {"restorecon_recursive",    {1,     kMax, do_restorecon_recursive}},
    //定义rc文件中支援的命令restorecon/restorecon_recursive及对应的实现函数
...

static int do_restorecon(const std::vector<std::string>& args) {
    int ret = 0;

    struct flag_type {const char* name; int value;};
    static const flag_type flags[] = {
        {"--recursive", SELINUX_ANDROID_RESTORECON_RECURSE},
        {"--skip-ce", SELINUX_ANDROID_RESTORECON_SKIPCE},
        {"--cross-filesystems", SELINUX_ANDROID_RESTORECON_CROSS_FILESYSTEMS},
        {0, 0}
    };
    //在rc命令中，增加支援的三个参数。注意，可以看出其并不支援force reset参数。

    int flag = 0;

    bool in_flags = true;
    for (size_t i = 1; i < args.size(); ++i) {
        if (android::base::StartsWith(args[i], "--")) {
            if (!in_flags) {
                LOG(ERROR) << "restorecon - flags must precede paths";
                return -1;
            }
            bool found = false;
            for (size_t j = 0; flags[j].name; ++j) {
                if (args[i] == flags[j].name) {
                    flag |= flags[j].value;
                    found = true;
                    break;
                }
            }
            if (!found) {
                LOG(ERROR) << "restorecon - bad flag " << args[i];
                return -1;
            }
        } else {
            in_flags = false;
            if (restorecon(args[i].c_str(), flag) < 0) {    //调用restorecon函数进行restore label过程。
                ret = -errno;
            }
        }
    }
    return ret;
}

static int do_restorecon_recursive(const std::vector<std::string>& args) {
    std::vector<std::string> non_const_args(args);
    non_const_args.insert(std::next(non_const_args.begin()), "--recursive"); //相比restorecon命令而言，只是默认多传入一个"--recursive"参数而已，后面流程完全一样。
    return do_restorecon(non_const_args);
}

```

上面restorecon函数的最终实现实在如下位置：

```
/* File name : external/selinux/libselinux/src/android/android.c */
static int selinux_android_restorecon_common(const char* pathname_orig,
                                             const char *seinfo,
                                             uid_t uid,
                                             unsigned int flags)
{
    bool nochange = (flags & SELINUX_ANDROID_RESTORECON_NOCHANGE) ? true : false;
    bool verbose = (flags & SELINUX_ANDROID_RESTORECON_VERBOSE) ? true : false;
    bool recurse = (flags & SELINUX_ANDROID_RESTORECON_RECURSE) ? true : false;
    bool force = (flags & SELINUX_ANDROID_RESTORECON_FORCE) ? true : false;
    bool datadata = (flags & SELINUX_ANDROID_RESTORECON_DATADATA) ? true : false;
    bool skipce = (flags & SELINUX_ANDROID_RESTORECON_SKIPCE) ? true : false;
    bool cross_filesystems = (flags & SELINUX_ANDROID_RESTORECON_CROSS_FILESYSTEMS) ? true : false;
    
    //可见selinux  lib中能支援的参数远比工具包或rc命令中所支援的参数多得多。我们可以根据实际情况修改init或toybox中的restorecon源码，以支援相互之间缺失的参数。
    
    
    bool issys;
    bool setrestoreconlast = true;
    struct stat sb;
    struct statfs sfsb;
    FTS *fts;
    FTSENT *ftsent;
    char *pathname = NULL, *pathdnamer = NULL, *pathdname, *pathbname;
    char * paths[2] = { NULL , NULL };
    int ftsflags = FTS_NOCHDIR | FTS_PHYSICAL;
    int error, sverrno;
    char xattr_value[FC_DIGEST_SIZE];
    ssize_t size;

    if (!cross_filesystems) {
        ftsflags |= FTS_XDEV;
    }

    if (is_selinux_enabled() <= 0)  //若selinux关闭，则后续动作均不再执行。
        return 0;

    __selinux_once(fc_once, file_context_init);
     //加载编译进系统的selinux file_contexts，并调用compute_file_contexts_hash函数计算预设载入系统中的file_contexts hash值。目的是为了后续检查用户是否通过fota或其他方式更新了修改过file_contexts的image。

    if (!fc_sehandle)
        return 0;

    /*
     * Convert passed-in pathname to canonical pathname by resolving realpath of
     * containing dir, then appending last component name.
     */
    pathbname = basename(pathname_orig);
    if (!strcmp(pathbname, "/") || !strcmp(pathbname, ".") || !strcmp(pathbname, "..")) {
        pathname = realpath(pathname_orig, NULL);
        if (!pathname)
            goto realpatherr;
    } else {
        pathdname = dirname(pathname_orig);
        pathdnamer = realpath(pathdname, NULL);
        if (!pathdnamer)
            goto realpatherr;
        if (!strcmp(pathdnamer, "/"))
            error = asprintf(&pathname, "/%s", pathbname);
        else
            error = asprintf(&pathname, "%s/%s", pathdnamer, pathbname);
        if (error < 0)
            goto oom;
    }

    paths[0] = pathname;
    issys = (!strcmp(pathname, SYS_PATH)
            || !strncmp(pathname, SYS_PREFIX, sizeof(SYS_PREFIX)-1)) ? true : false;
//检查传入路径是否为/sys目录

    if (!recurse) { //若未传入-R或-recursive参数(无需遍历)，则检查随后传入的文件路径参数是否有问题。
        if (lstat(pathname, &sb) < 0) {
            error = -1;
            goto cleanup;
        }

        error = restorecon_sb(pathname, &sb, nochange, verbose, seinfo, uid);
        //进行单文件(夹)label动作。
        goto cleanup;
    }


//此后的动作是recursive模式下的逻辑。

    /*
     * Ignore restorecon_last on /data/data or /data/user
     * since their labeling is based on seapp_contexts and seinfo
     * assignments rather than file_contexts and is managed by
     * installd rather than init.
     */
    if (!strncmp(pathname, DATA_DATA_PREFIX, sizeof(DATA_DATA_PREFIX)-1) ||
        !strncmp(pathname, DATA_USER_PREFIX, sizeof(DATA_USER_PREFIX)-1) ||
        !strncmp(pathname, DATA_USER_DE_PREFIX, sizeof(DATA_USER_DE_PREFIX)-1) ||
        !fnmatch(EXPAND_USER_PATH, pathname, FNM_LEADING_DIR|FNM_PATHNAME) ||
        !fnmatch(EXPAND_USER_DE_PATH, pathname, FNM_LEADING_DIR|FNM_PATHNAME))
        setrestoreconlast = false;

    /* Also ignore on /sys since it is regenerated on each boot regardless. */
    if (issys)
        setrestoreconlast = false;

    /* Ignore files on in-memory filesystems */
    if (statfs(pathname, &sfsb) == 0) {
        if (sfsb.f_type == RAMFS_MAGIC || sfsb.f_type == TMPFS_MAGIC)
            setrestoreconlast = false;
    }
    
    //前面有将，此处将sys文件路径、ranfs文件路径和前面提到过的/data/下一系列路径进行标记，对于这些文件路径而言，将不判断file_contexts的hash值是否有变而直接进入后续逻辑。（对其余文件夹，将计算文件夹中存入的file_contexts hash值是否和限制系统中file_contexts的hash一致，若user烧了更换过file_contexts rule的image，则直接按照新rule进行遍历及relabel）

    if (setrestoreconlast) {
     //对于非 /sys、ramfs和data特定目录的文件路径，将进行进一步检查，来确定recursive动作是否真要要执行。
        size = getxattr(pathname, RESTORECON_LAST, xattr_value, sizeof fc_digest);
        //通过getxattr函数，读取所传入文件node中"security.restorecon_last"属性值，该值对应的数据是上次restorecon执行过程中所存入的file_contexts hash值。
        if (!force && size == sizeof fc_digest && memcmp(fc_digest, xattr_value, sizeof fc_digest) == 0) {
        //若不加入force命令，且当前系统的file_contexts hash值与文件node中存入的hash值完全一致（说明系统未更新过sepolicy），则不再进行重复relabel动作，以保证后续操作系统对系统文件selabel所做的改动不被无意义地改变。（这里我们可以通过烧入修改过sepolicy的boot image并重启手机，让已经label过得部分文件再次relabel）
            selinux_log(SELINUX_INFO,
                        "SELinux: Skipping restorecon_recursive(%s)\n",
                        pathname);
            error = 0;
            goto cleanup;
        }
    }

    fts = fts_open(paths, ftsflags, NULL);
    if (!fts) {
        error = -1;
        goto cleanup;
    }

    error = 0;
    while ((ftsent = fts_read(fts)) != NULL) {
        switch (ftsent->fts_info) {
        case FTS_DC:   //A directory that causes a cycle in the tree. 
            selinux_log(SELINUX_ERROR,
                        "SELinux:  Directory cycle on %s.\n", ftsent->fts_path);
            errno = ELOOP;
            error = -1;
            goto out;
        case FTS_DP: //A  directory  being  visited  in  postorder.
            continue;
        case FTS_DNR: //A directory which cannot be read.
            selinux_log(SELINUX_ERROR,
                        "SELinux:  Could not read %s: %s.\n", ftsent->fts_path, strerror(errno));
            fts_set(fts, ftsent, FTS_SKIP);
            continue;
        case FTS_NS: //A file for which no stat(2) information was available.
            selinux_log(SELINUX_ERROR,
                        "SELinux:  Could not stat %s: %s.\n", ftsent->fts_path, strerror(errno));
            fts_set(fts, ftsent, FTS_SKIP);
            continue;
        case FTS_ERR: // This is an error return.
            selinux_log(SELINUX_ERROR,
                        "SELinux:  Error on %s: %s.\n", ftsent->fts_path, strerror(errno));
            fts_set(fts, ftsent, FTS_SKIP);
            continue;
        case FTS_D:  //A directory being visited in preorder.
            if (issys && !selabel_partial_match(fc_sehandle, ftsent->fts_path)) {
                fts_set(fts, ftsent, FTS_SKIP);
                continue;
            }

            if (skipce &&
                (!strncmp(ftsent->fts_path, DATA_SYSTEM_CE_PREFIX, sizeof(DATA_SYSTEM_CE_PREFIX)-1) ||
                 !strncmp(ftsent->fts_path, DATA_MISC_CE_PREFIX, sizeof(DATA_MISC_CE_PREFIX)-1))) { //若restorecon带入了--recursive和--skip-ce参数，则跳过/data/misc_ce/和/data/system_ce/两个ce目录的relabel过程。
                // Don't label anything below this directory.
                fts_set(fts, ftsent, FTS_SKIP);
                // but fall through and make sure we label the directory itself
            }

            if (!datadata &&
                (!strcmp(ftsent->fts_path, DATA_DATA_PATH) ||
                 !strncmp(ftsent->fts_path, DATA_USER_PREFIX, sizeof(DATA_USER_PREFIX)-1) ||
                 !strncmp(ftsent->fts_path, DATA_USER_DE_PREFIX, sizeof(DATA_USER_DE_PREFIX)-1) ||
                 !fnmatch(EXPAND_USER_PATH, ftsent->fts_path, FNM_LEADING_DIR|FNM_PATHNAME) ||
                 !fnmatch(EXPAND_USER_DE_PATH, ftsent->fts_path, FNM_LEADING_DIR|FNM_PATHNAME))) {
                 //若未传入SELINUX_ANDROID_RESTORECON_DATADATA flag（即restorecon工具的-D的参数），则默认跳过上节提到的/data/目录内一系列目录的relabel过程。
                // Don't label anything below this directory.
                fts_set(fts, ftsent, FTS_SKIP);
                // but fall through and make sure we label the directory itself
            }
            /* fall through */
        default:
            error |= restorecon_sb(ftsent->fts_path, ftsent->fts_statp, nochange, verbose, seinfo, uid);//对于其余的正常文件路径，调用restorecon_sb函数进行正常的restorecon动作。
            break;
        }
    }

    // Labeling successful. Mark the top level directory as completed.
    if (setrestoreconlast && !nochange && !error)
        setxattr(pathname, RESTORECON_LAST, fc_digest, sizeof fc_digest, 0);
        //将最新的系统sepolicy hash值，写入relabel过的文件路径根目录node中，以便后续不在进行多余的relabel动作。

out:
    sverrno = errno;
    (void) fts_close(fts);
    errno = sverrno;
cleanup:
    free(pathdnamer);
    free(pathname);
    return error;
oom:
    sverrno = errno;
    selinux_log(SELINUX_ERROR, "%s:  Out of memory\n", __FUNCTION__);
    errno = sverrno;
    error = -1;
    goto cleanup;
realpatherr:
    sverrno = errno;
    selinux_log(SELINUX_ERROR, "SELinux: Could not get canonical path for %s restorecon: %s.\n",
            pathname_orig, strerror(errno));
    errno = sverrno;
    error = -1;
    goto cleanup;
}
```

需要补充的是，使用setxattr函数向文件夹node中存入的hash值，其实是存到了真正的文件系统中，并不会随着关机或重启而发生变化。这样就让我们有机会通过判断用户是否更新了带新file_contexts rule的image，来决定在调用restorecon_recursive时，对于未修改过sepolicy rule的文件节点就不再重复进行relabel动作了。

##  饭后甜点

带来一个经典的案例作为饭后甜点：

> 一位user从Android O Fota退版回Android N,此后再次想进行Fota升级
却发现系统卡在recovery mode再也无法出来。即使此时强按power key进行重启，依然会自动进入recovery mode。
> 
> 值得注意的是：其他user均未回报此问题，即该问题或许只有在特殊的操作手法下才能复现。

这里先直接给出debug后的结论：

>  在Android N 的img中，开始fota并进入recovery image后，主进程会试图读取adc分区中的log文件，以便将recovery过程的关键log存入其中。但该机器的log文件selinux label却是”u:object_r:log_file:s0“，故permission denied，系统逻辑直接跳出，故卡死在recovery中。另外，由于misc分区的command和cache分区的command都未被清除，所以再次开机自然也会进入recovery mode。

这里我们首先要弄清楚两个问题： 
```
1. 为何recovery mode 中必须打开abc分区的log文件并且打开失败就直接退出？

为了方便debug，RD将recovery过程中的一部分log冲定向到abc分区，以便能永久保存避免误删。
而打开失败就退出，应该是coding bug。

2. log文件的sepolicy到底为何改变了？

在Android N中，我们未对adc分区做特别的selinux label动作，故abc分区中的所有文件默认都是unlabeled sepolicy权限。而Android O中，我们有针对对adc分区加入了 ”u:object_r:log_file:s0“  label以便对log权限进性规范化统一。 

问题机器在Android N的recovery中，查看abc分区内所有文件，发现其他文件都是unlabeled状态，唯独log文件是log_file状态。而同样操作后，正常机器内的abc分区文件label 都是unlabeled。

所以该文件的selinux label改变必有玄机。而该玄机就是mount时的强行加入selabel导致。

```
大家知道，若在rc文件中mount分区时
> mount ext4 /dev/block/bootdevice/by-name/asdf /abc rw noatime nosuid nodev barrier=1 data=ordered context=u:object_r:log_file:s0

直接加入context=“*”参数,则分区mount后，默认文件夹及根目录将默认被植入log_file的selabel，且该selabel其实只是记录在ram中，并未实际写入文件系统。但若此时新建一个文件，则新建文件的selabel却真真正正地被写入了文件系统中。

这就带来一个问题，若以不加context=“*”参数的方式再次mount该分区，则前面新建的文件由于selabel被写入文件系统，故将会保留log_file的selabel，而其余文件会变成unlabeled的selabel。

前面用户遇到的无法fota问题就是因为log文件是在Android O中新建出来的(Android N中没有进入过recovery mode，而是直接用fastboot进行的刷机)，而正常用户都是fota升级，所以在Android N中已经新建出了log文件。这样在Android O中以“context=u:object_r:log_file:s0”的方式mount abc分区的结果是，所有在Android N中存在过的文件均被强行label成log_file到ram中，故不论Android N中文件的label怎样及是否进行新的文件创建(写入实际文件系统)，均不影响Android O的selinux权限。

但当用户将系统回退到Android N的recovery image后，由于Android N未进行mount加context=“*”参数的动作，故在Android O中新建立的文件label(实际写入文件系统中)就显现出正式的模样，即log文件的label变成了log_file的label。

补充一个问题，**为何只有新建的文件，其selinux的label才会被写入文件系统呢？**

问题的答案在restorecon函数的实现中可以找到端倪：
```
static int restorecon_sb(const char *pathname, const struct stat *sb,
                         bool nochange, bool verbose,
                         const char *seinfo, uid_t uid)
{
    char *secontext = NULL;
    char *oldsecontext = NULL;
    int rc = 0;

    if (selabel_lookup(fc_sehandle, &secontext, pathname, sb->st_mode) < 0)
        return 0;  /* no match, but not an error */

    if (lgetfilecon(pathname, &oldsecontext) < 0)
        goto err;

   ...
   
    if (strcmp(oldsecontext, secontext) != 0) {
    
    //若系统发现现有的selinux label已经符合预期(mount时加参数强行label导致)，则不在进行写入实际文件系统的后续动作。所以对于新建的文件而言，其值将会写入文件系统中而不仅仅存在与ram。
        if (verbose)
            selinux_log(SELINUX_INFO,
                        "SELinux:  Relabeling %s from %s to %s.\n", pathname, oldsecontext, secontext);
        if (!nochange) {
            if (lsetfilecon(pathname, secontext) < 0)
                goto err;
        }
    }

    rc = 0;

out:
    freecon(oldsecontext);
    freecon(secontext);
    return rc;

err:
    selinux_log(SELINUX_ERROR,
                "SELinux: Could not set context for %s:  %s\n",
                pathname, strerror(errno));
    rc = -1;
    goto out;
}
```



所以建议大家将selinux的label定义到file_contexts中以便进行更一般化的管理，尽量避免在mount文件系统的时候加入context=“*”参数，这样很容易麻痹自己和同事，造成不必要的兼容性问题。 故再次强调，请千万慎用mount时候加如context=“*”以求打入label的做法。
