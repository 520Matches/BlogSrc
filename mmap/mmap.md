# linux中mmap详解
- 作者：*Match*
## mmap是什么东东？
- 在Linux内核中**mmap**是内存映射，它可以把内核空间中的内存的物理地址空间映射到用户空间。
## mmap有什么作用？
- 好啦，知道了**mmap**是什么东东以后，小伙伴们肯定想知道这玩意能用来干啥？下面通过一个简单的例子来说明一下。
- *例子*：现在我有一块显卡，显卡会有自己的显存（一般是比较大的），如果Linux内核要使用这块显卡，那么内核中应该有显卡的驱动程序，驱动程序会把用户程序输入的内容复制到内核的内存中（显卡的地址），在内核中会调用copy_from_user()这个函数。这样做基本功能是能实现，但是，我要说但是，这样做每次读写显存的时候都要从内核和用户空间的数据copy来copy去，严重影响的效率，没有好好发挥显卡的功能。那么这个时候，**mmap**就登场了，它可以把内核中的显存映射到用户空间，这样用户程序就能直接对显存进行操作了，省去了copy的操作，效率高了很多。没错这就是**mmap**的厉害之处了，同时显存有了两份映射，一份在内核空间，一份在用户空间，用户和内核都能对其进行操作，perfect。
![图1](D:/images/mmap.png)
## 简单写一波代码试试
#### 驱动程序
- 初始化
```
//生成设备号
dev = MKDEV(VFB_MAJOR,VFB_MINOR);
//注册字符设备
ret = register_chrdev_region(dev, VFB_DEV_CNT,VFB_DEV_NAME);

//初始化cdev
cdev_init(&vfbdev.cdev,&vfb_fops);
vfbdev.cdev.owner = THIS_MODULE;
//把设备添加到cdev_map这个散列表中
ret = cdev_add(&vfbdev.cdev,dev,VFB_DEV_CNT);
//获取空闲内存空间(如果没有空闲内存空间GFP_KERNEL参数会使进程阻塞)
addr = __get_free_page(GFP_KERNEL);
```

- mmap驱动代码实现
```
...
//调用内存映射函数
remap_pfn_range(vma,vma->vm_start,virt_to_phys(vfbdev.buf) >> PAGE_SHIFT,vma->vm_end - vma->vm_start,vma->vm_page_prot);
...
```
#### 用户程序
- 把26个英文字母写入到显存中
```
//必须先用mknod命令创建检点vfb，不然会报打开文件失败的错
fd = open("/dev/vfb",O_RDWR);
//直接使用mmap函数，调用这个函数以后会去调用驱动代码中的remap_pfn_range函数来完成内存映射。
start = mmap(NULL,32,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
//然后就是so easy啦，直接在获取到的内存地址赋值就可以了。
for(i=0;i < 26;i++)
{
	*(start + i) = i + 'a';
}
*(start + i) = '\0';
```
## 结尾
- 完整代码地址 **git clone https://github.com/520Matches/CoderStudy.git**
- 有不对的地方欢迎指出 **HengDi_Sun@163.com**
- 喜欢的朋友关注微信订阅号 **CoderPark**