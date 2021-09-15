+++
title = "Linux内核模块编写之？: Platform Device"
date = 2021-09-14T20:34:06+08:00
draft = true
[[copyright]]
  owner = "贺若舟"
  date = "2021"
  license = "cc-by-nd-4.0"
+++

> 警告
> 
> 描述结构体字段的记号使用C++风格`a::b`。其实用`a.b`也可以，但我不喜欢。
>

## Platform Device

在一个系统中，很多设备并不在一个提供了枚举、热拔插、为每个设备提供唯一识别码的总线上，无法被发现和识别。比方说片上设备和i2c、SPI总线上的设备，或者由GPIO直接驱动的设备。我们希望这些设备也能作为Linux设备驱动模型的一部分。所以Linux引入了platform device机制。

一个platform device必须要用platform driver驱动。下面就来详细介绍他们。

## Platform Driver的实现

platform device的驱动需要实现一个`platform_driver`结构体。该结构体的源码如下：

```c
struct platform_driver {
	int (*probe)(struct platform_device *);     // 驱动加载时调用此函数
	int (*remove)(struct platform_device *);    // 驱动卸载时调用此函数
	void (*shutdown)(struct platform_device *); // 关闭设备
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
	bool prevent_deferred_probe;
};
```

这个结构体继承了`device_driver`。你可能会问怎么搞的函数多态继承，这不C++才有的吗？没错，C里面没有魔法。都是要用人力完成的：

```c
/**
 * __platform_driver_register - register a driver for platform-level devices
 * @drv: platform driver structure
 * @owner: owning module/driver
 */
int __platform_driver_register(struct platform_driver *drv,
							struct module *owner)
{
	drv->driver.owner = owner;
	drv->driver.bus = &platform_bus_type;
	drv->driver.probe = platform_drv_probe;        // 手动“重载”了device_driver里的函数们，
	drv->driver.remove = platform_drv_remove;      // 于是在device_driver里调用这些函数，
	drv->driver.shutdown = platform_drv_shutdown;  // 也不会有错误出现。

	return driver_register(&drv->driver);
}
EXPORT_SYMBOL_GPL(__platform_driver_register);
```

所以实现platform driver的时候，这些硬件操纵函数直接写在`platform_driver`结构体里，不要写在`device_driver`里。另外一般还需要填写的是`platform_driver::driver::name`，作为platform driver驱动的名称，并使用`platform_driver::driver::of_match_table`登记兼容名称。

## Platform Driver的使用

因为platform device那样的设备是无法被动态检测的，故需要进行静态描述：

- 使用内核代码描述（Legacy）
- 使用设备树文件描述
- BIOS ACPI表（一般仅在PC机上用这种方式）

### 内核代码（Legacy）

使用`platform_device_register()`函数传入一个实现好的`platform_device`结构体，这样就能注册一个platform设备。有多个platform设备的话，可以使用`platform_add_devices()`函数。

先来看看`platform_device`里面有什么：

```c
struct platform_device {
	const char		*name; // 驱动名称，必须和platform_driver的对应
	int				id;    // id号
	bool			id_auto;
	struct device	dev;   // device结构体
	u64				platform_dma_mask;
	struct device_dma_parameters dma_parms;
	u32				num_resources; // 资源数组长度
	struct resource	*resource;     // 资源数组

	const struct platform_device_id	*id_entry;
	char *driver_override; /* Driver name to force a match */

	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;

	/* arch specific additions */
	struct pdev_archdata	archdata;
};
```

一般来说，需要提前定义好`resource`/`num_resources`和`dev.platform_data`，才能进行注册。但也需要具体情况具体分析。

这次演示的是使用代码将`leds-gpio`模块应用上我买的三原色LED灯。该LED灯原理图为：

![rgb-leds-circuit](/~fward/images/rgb-leds-circuit.png)

可以看到是个共阴极电路。R、G、B接口设为高电平就能让二极管发光。这是个很适合GPIO驱动的一个电器原件。我把它三个脚分别接在Orange Pi Zero的PA10，PA13，PA2上：

```c
#define red_led_gpio   10
#define green_led_gpio 13
#define blue_led_gpio  2
```

使用任何现成的platform driver模块前，首先应该看看driver模块内部怎么进行的初始化设置。所以需要查看他的`probe`函数。

```c
static struct platform_driver gpio_led_driver = {
	.probe		= gpio_led_probe,
	.shutdown	= gpio_led_shutdown,
	.driver		= {
		.name	= "leds-gpio",
		.of_match_table = of_gpio_leds_match,
	},
};
```

他的`probe`函数是`gpio_led_probe()`。来看看这个函数干了什么：

```c
static int gpio_led_probe(struct platform_device *pdev)
{
	struct gpio_led_platform_data *pdata = dev_get_platdata(&pdev->dev);
	struct gpio_leds_priv *priv;
	int i, ret = 0;
	// ...
	// 中略，内容为根据pdata构造priv
	// (仅在pdata有效时，否则走设备树流程)
	// ...
	platform_set_drvdata(pdev, priv);
	return 0;
}
```

这里可以看到，一开始这个probe函数就用`dev_get_platdata()`取出`pdev->dev`的`platform_data`。之后就开始了初始化`pdev->dev`的`driver_data`字段的过程。从始至终只使用了`device::platform_data`作为初始化的数据。所以我们要使用这个platform driver模块，只需要在`platform_device`填入合适的`platform_device::device::platform_data`就可以了。

下面是完整代码：

```c
#include <linux/module.h>           // 所有模块都需要
#include <linux/moduleparam.h>      // 模块参数
#include <linux/kernel.h>           // printk和KERN_INFO等等
#include <linux/init.h>             // __init、__exit的定义
#include <linux/platform_device.h>  // platform_device的定义
#include <linux/device.h>           // device的定义
#include <linux/leds.h>             // gpio_led_platform_data等等结构体的定义

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Fw[a]rd");
MODULE_DESCRIPTION("A rgb leds module");

#define red_led_gpio   10
#define green_led_gpio 13
#define blue_led_gpio  2

static struct gpio_led rgb_leds_info[3] = {
	{.name = "red", .gpio = red_led_gpio, .active_low = 0u},
	{.name = "green", .gpio = green_led_gpio, .active_low = 0u},
	{.name = "blue", .gpio = blue_led_gpio, .active_low = 0u},
};


static struct gpio_led_platform_data rgb_leds_data = {
	.num_leds = 3,
	.leds = rgb_leds_info,
};

// 防止报错用
void rgb_leds_device_release(struct device *dev) {
	// do nothing
	return;
}

// platform设备结构体
static struct platform_device rgb_leds_device =
{
	.name = "leds-gpio", // 必须是这个名字，否则无法加载
	.id = 0x2233,
	.dev = {
		.platform_data = &rgb_leds_data,
		.release = &rgb_leds_device_release,
	}
};


// 加载模块调用的函数
static int __init rgb_init(void)
{
    int res = platform_device_register(&rgb_leds_device);
	printk(KERN_INFO "rgb_leds: registered, return %d\n", res);
    return 0;
}

// 卸载模块调用的函数，需要用__exit宏
static void __exit rgb_exit(void)
{   
    platform_device_unregister(&rgb_leds_device);
    printk(KERN_INFO "rgb_leds: Goodbye!\n");
}

// 定义模块的加载卸载函数
module_init(rgb_init);
module_exit(rgb_exit);
```

代码里可以发现我除了定义`platform_data`外，还多定义了个`release`函数，现在没有定义`device::release()`的话会有警告：

```
Device 'leds-gpio.8755' does not have a release() function, it is broken and must be fixed. See Documentation/core-api/kobject.rst.
```

下面就来看看效果：

```
root@opi:/sys/class/leds # ll
total 0
lrwxrwxrwx 1 root root 0 Oct 13 12:28 blue -> ../../devices/platform/leds-gpio.8755/leds/blue
lrwxrwxrwx 1 root root 0 Oct 13 12:28 green -> ../../devices/platform/leds-gpio.8755/leds/green
lrwxrwxrwx 1 root root 0 Oct 13 12:19 orangepi:green:pwr -> ../../devices/platform/leds/leds/orangepi:green:pwr
lrwxrwxrwx 1 root root 0 Oct 13 12:19 orangepi:red:status -> ../../devices/platform/leds/leds/orangepi:red:status
lrwxrwxrwx 1 root root 0 Oct 13 12:28 red -> ../../devices/platform/leds-gpio.8755/leds/red
```

看来已经安装上了

```
root@opi:class/leds # echo 1 > blue/brightness
```

![rgb-leds-blue-on](/~fward/images/rgb-leds-blue-on.jpg)

没毛病

```
root@opi:/sys/class/leds # echo 1 > green/brightness
root@opi:/sys/class/leds # echo 1 > red/brightness
```

![rgb-leds-all-on](/~fward/images/rgb-leds-all-on.jpg)

测试成功了

你还可以玩些花的，这里就不演示了

```
root@opi:/sys/class/leds # echo heartbeat > green/trigger
```

### 设备树

上回说了怎么用内核代码驱动平台设备。接下来我们将对如何使用设备树进行介绍。

设备树（Device Tree）是描述设备的树结构文件。设备树一开始并非是Linux使用的设备描述方法，所以只能说是Linux兼容了这种方法，里面的字段名并不严格对应Linux的设备描述结构体里面的字段名。

那么设备树文件是怎么样被Linux内核分析和利用的呢？

通过分析内核日志文件可以发现内核使用fdt模块处理设备树。先是通过`of_flat_dt_match_machine()`函数，……（咕咕咕，还在写）

要使用设备树方法，仍然需要从`platform_driver::probe()`入手进行分析。我们来看看熟悉的`leds-gpio`的：

```c
static int gpio_led_probe(struct platform_device *pdev)
{
	struct gpio_led_platform_data *pdata = dev_get_platdata(&pdev->dev);
	struct gpio_leds_priv *priv;
	int i, ret = 0;

	if (pdata && pdata->num_leds) {
		// 中略，在pdata（gpio_led_platform_data*）有效时走常规路线
		// ...
	} else { // 这里是设备树路线
		priv = gpio_leds_create(pdev); // 通过设备树内容构造priv
		if (IS_ERR(priv))
			return PTR_ERR(priv);
	}

	platform_set_drvdata(pdev, priv);

	return 0;
}
```

看来还需要看看通过设备树内容构造priv的函数`gpio_leds_create()`了：

```c
static struct gpio_leds_priv *gpio_leds_create(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct fwnode_handle *child;
	struct gpio_leds_priv *priv;
	int count, ret;

	count = device_get_child_node_count(dev); // 获取子节点个数
	if (!count)
		return ERR_PTR(-ENODEV);

	priv = devm_kzalloc(dev, struct_size(priv, leds, count), GFP_KERNEL);  // 给priv分配内存空间
	if (!priv)
		return ERR_PTR(-ENOMEM);

	device_for_each_child_node(dev, child) { // 每个子节点进行foreach循环
		struct gpio_led_data *led_dat = &priv->leds[priv->num_leds];
		struct gpio_led led = {};
		const char *state = NULL;

		/*
		 * Acquire gpiod from DT with uninitialized label, which
		 * will be updated after LED class device is registered,
		 * Only then the final LED name is known.
		 */
		led.gpiod = devm_fwnode_get_gpiod_from_child(dev, NULL, child,
							     GPIOD_ASIS,
							     NULL);
		if (IS_ERR(led.gpiod)) {
			fwnode_handle_put(child);
			return ERR_CAST(led.gpiod);
		}

		led_dat->gpiod = led.gpiod;

		fwnode_property_read_string(child, "linux,default-trigger",
					    &led.default_trigger);

		if (!fwnode_property_read_string(child, "default-state",
						 &state)) {
			if (!strcmp(state, "keep"))
				led.default_state = LEDS_GPIO_DEFSTATE_KEEP;
			else if (!strcmp(state, "on"))
				led.default_state = LEDS_GPIO_DEFSTATE_ON;
			else
				led.default_state = LEDS_GPIO_DEFSTATE_OFF;
		}

		if (fwnode_property_present(child, "retain-state-suspended"))
			led.retain_state_suspended = 1;
		if (fwnode_property_present(child, "retain-state-shutdown"))
			led.retain_state_shutdown = 1;
		if (fwnode_property_present(child, "panic-indicator"))
			led.panic_indicator = 1;

		ret = create_gpio_led(&led, led_dat, dev, child, NULL);
		if (ret < 0) {
			fwnode_handle_put(child);
			return ERR_PTR(ret);
		}
		/* Set gpiod label to match the corresponding LED name. */
		gpiod_set_consumer_name(led_dat->gpiod,
					led_dat->cdev.dev->kobj.name);
		priv->num_leds++;
	}

	return priv;
}
```

咕咕咕，施工中…

<!-- 
```dts
    leds {
        compatible = "gpio-leds";

        pwr_led {
            label = "orangepi:green:pwr";
            gpios = <0x3a 0x00 0x0a 0x00>;
            default-state = "on";
        };   

        status_led {
            label = "orangepi:red:status";
            gpios = <0x0c 0x00 0x11 0x00>;
        };   
    };
``` -->