

## Linux API


### 内存分配
#### devm_kzalloc / devm_kfree
函数`devm_kzalloc()` 和 `kzalloc()`一样都是内核内存分配函数，但是`devm_kzalloc()`是跟设备(device)有关的，当设备(device)被detached或者驱动(driver)卸载unloaded时，内存会被自动释放。<br>另外，当内存不在使用时，可以使用函数`devm_kfree()`释放。

### platform device
#### platform_set_drvdata / platform_get_drvdata
Linux 设备驱动的 platform 框架中，<br>在驱动的 probe 函数中获取 device 的相关信息，使用 `platform_set_drvdata`进行保存。<br>在其他地方使用 `platform_get_drvdata` 进行获取。

### device
#### dev_name / dev_set_name
在 2.6.35 后的内核中`struct device`已经没有`bus_id`成员，取而代之的是通过`dev_name`和`dev_set_name`对设备的名字进行操作。

### dts 解析
#### of_platform_bus_create
根据 dts 创建对应的 device，并添加到 kernel 中
#### of_property_read_u32(np, "rk,xxx", &yyy)
读取`"rk,xxx"`这个属性的值，并赋值给`yyy`。
#### of_parse_phandle
通过 phandle 引用 node。
```
struct device_node *of_parse_phandle(struct device_node *np,const char *phandle_name, int index)
```
#### 