# RK Clock 查看与操作

Kernel Version：Linux4.4


## 查询

```
cat /sys/kernel/debug/clk/clk_summary

```

## 修改
打开 CONFIG_ROCKCHIP_PM_TEST
```
cat /sys/pm_tests/clk_rate

#get clk rate:
echo get [clk_name] > /sys/pm_tests/clk_rate
#set clk rate:
echo rawset [clk_name] [rate(Hz)] > /sys/pm_tests/clk_rate
#enable clk:
echo open [clk_name] > /sys/pm_tests/clk_rate
#disable clk:
echo close [clk_name] > /sys/pm_tests/clk_rate
```