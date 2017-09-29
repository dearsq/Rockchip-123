# API sysfs

参考文档 `./Documentation/filesystems/sysfs.txt`
## 是什么

sysfs 是用于表现设备驱动模型的文件系统，它基于ramfs。它提供了一种导出内核数据结构、属性、和用户空间之间联系 的方法。

sysfs 中提供了四类文件的创建与管理，分别是目录、普通文件、软链接文件、二进制文件。<br>**目录**层次往往代表着设备驱动模型的**结构**，**软链接**文件则代表着不同部分间的**关系**。比如某个设备的目录只出现在/sys/devices下，其它地方涉及到它时只好用软链接文件链接过去，**保持了设备唯一的实例**。<br>而**普通文件**和**二进制文件**往往代表了设备的**属性**，读写这些文件需要调用相应的属性读写。

sysfs 是表现设备驱动模型的文件系统，它的目录层次实际反映的是对象的层次。为了配合这种目录，linux专门提供了两个结构作为sysfs 的骨架，它们就是 struct kobject 和 struct kset。<br>sysfs 是完全虚拟的，它的每个目录其实都对应着一个kobject，要想知道这个目录下有哪些子目录，就要用到 kset。从面向对象的角度来讲，kset 继承了 kobject 的功能，既可以表示sysfs 中的一个目录，还可以包含下层目录。对于kobject和kset，会在其它文章中专门分析到，这里简单描述只是为了更好地介绍 sysfs 提供的API。

sysfs 模块提供给外界的接口完全展现在 ` include/linux/sysfs.h`中。

## Attributes 属性
**属性**可以以 文件系统中的常规文件的 kobjects 的形式被导出。
Sysfs 发送文件 IO 操作给这些为属性而定义的方法。提供了一个工具来读写内核**属性**。

属性应该是 ASCII码写的文本文件，最好只有一个值。
不过这有可能不是最有效的，所以如果是某种类型的值的数组，也是可以的。

```c
//属性的通用结构
struct attribute {  
  const char      *name;  
  struct module   *owner;  
  mode_t          mode;  
};

//一组属性的集合
struct attribute_group {  
  const char  *name;  
  mode_t      (*is_visible)(struct kobject *, struct attribute *, int);  
  struct attribute    **attrs;  
};

//二进制属性的通用结构
struct bin_attribute {  
  struct attribute    attr;  
  size_t          size;  
  void            *private;  
  ssize_t (*read)(struct kobject *, struct bin_attribute *,  
          char *, loff_t, size_t);  
  ssize_t (*write)(struct kobject *, struct bin_attribute *,  
           char *, loff_t, size_t);  
  int (*mmap)(struct kobject *, struct bin_attribute *attr,  
          struct vm_area_struct *vma);  
};  

//静态初始化属性的宏
define __ATTR(_name,_mode,_show,_store) { \  
  .attr = {.name = __stringify(_name), .mode = _mode },   \  
  .show   = _show,                    \  
  .store  = _store,                   \  
}  
#define __ATTR_RO(_name) { \  
  .attr   = { .name = __stringify(_name), .mode = 0444 }, \  
  .show   = _name##_show,                 \  
}  
#define __ATTR_NULL { .attr = { .name = NULL } }  
#define attr_name(_attr) (_attr).attr.name
```

### sysfs_ops
sysfs_ops 包含 show 和 store 两个函数指针，分别在 sysfs 文件读和写时调用
```c
struct sysfs_ops {  
  ssize_t (*show)(struct kobject *, struct attribute *,char *);  
  ssize_t (*store)(struct kobject *,struct attribute *,const char *, size_t);  
};  
```

### sysfs 其他的 API
```c
//创建工作队列，稍后调用 func。
sysfs_schedule_callback()
应用：通过工作队列回调的方式删除属性文件。（属性文件读写函数无法删除属性文件或者 kobject 目录，因为调用函数时是加锁的，删除也需要加锁）
int sysfs_schedule_callback(struct kobject *kobj, void (*func)(void *),void *data, struct module *owner);  
//创建一个kobject对应的目录，目录名就是kobj->name。
sysfs_create_dir()
//删除kobj对应的目录。删除一个目录也会相应地删除目录下的文件及子目录。
sysfs_remove_dir()
//修改kobj对应目录的名称。
sysfs_rename_dir()
//将kobj对应的目录移到new_parent_kobj对应的目录下。
sysfs_move_dir()
```