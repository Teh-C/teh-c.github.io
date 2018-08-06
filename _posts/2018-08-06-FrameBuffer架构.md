---
layout: post
title: 嵌入式Linux驱动 - FrameBuffer架构
date: 2018-08-06 10:52 
tags: 嵌入式Linux 
---
# FrameBuffer架构

-------------------
&emsp;&emsp;Linux内核中，FrameBuferr设备驱动的源码主要分布在linux/include/fb.h和linux/drivers/video/fbmem.c，它们处于FrameBuffer驱动体系结构中的中间层，它为上层的用户程序提供系统调用接口，也为底层特定硬件驱动提供了接口。

-------------------
1 FrameBuffer驱动程序的实现
--
&emsp;&emsp;从应用程序角度看，其通过内核对FrameBuffer的控制主要通过以下三种方式：
&emsp;&emsp;**(1) 读写/dev/fb相当于读/写屏幕缓冲区，对应驱动中的write和read操作。**
&emsp;&emsp;**(2) 通过映射操作，可将屏幕缓冲区的物理地址映射到用户空间的一段虚拟地址中，之后用户可以通过读写这段虚拟地址访问屏幕缓冲区，在屏幕上绘图。**
&emsp;&emsp;**(3) I/O控制对于帧缓冲设备，设备文件的icctl()函数可读取/设置显示设备及屏幕的参数，如分辨率、显示颜色数、屏幕大小等。ioctl()函数是由底层的驱动程序完成的。**
&emsp;&emsp;因此，驱动设计者完成FrameBuffer驱动的工作已经不多了，只需要分配显存的大小、初始化LCD控制器、设置修改硬件设备相应的**struct fb_fix_screeninfo**信息和**struct fb_var_screaninfo**信息。这些信息是与具体的显示设备相关的。
&emsp;&emsp;帧缓冲设备属于字符设备，采用“文件层-驱动层”的接口方式。在文件层次上，Linux为其定义了**fb_fops**结构体。

-------------------

2 FrameBuffer对应的数据结构体
--
&emsp;&emsp;FrameBuffer驱动程序有几个重要的数据结构，分别为：**fb_info**、**fb_ops**、**fb_cmap**、**fb_var_screeninfo**和**fb_fix_screeninfo**，这些结构体之间的关系图如下图所示：![这里写图片描述](https://img-blog.csdn.net/20180806102215550?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NhbnRhcGFzc2VyYnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
&emsp;&emsp;下面对这些数据结构进行详细的介绍。
## 2.1 fb_info结构体
&emsp;&emsp;**fb_info是帧缓冲FrameBuffer设备驱动中一个非常重要的结构体。它包含了驱动实现的底层函数和记录设备状态的数据。一个帧缓冲区对应一个fb_info结构体。这个结构体定义如下:**

```C++
struct fb_info {
	int node;
	int flags;
	struct mutex lock; /* 为open/release/ioctl接口提供互斥锁 */
	struct mutex mm_lock; /* 为fb_mmap and smem_* fields提供互斥锁 */
	struct fb_var_screeninfo var; /* Current var */
	struct fb_fix_screeninfo fix; /* Current fix */
	struct fb_monspecs monspecs; /* Current Monitor specs */
	struct work_struct queue; /* Framebuffer event queue */
	struct fb_pixmap pixmap; /* Image hardware mapper */
	struct fb_pixmap sprite; /* Cursor hardware mapper */
	struct fb_cmap cmap; /* Current cmap */
	struct list_head modelist; /* mode list */
	struct fb_videomode *mode; /* current mode */
// 如果配置了LCD支持背光
#ifdef CONFIG_FB_BACKLIGHT
	/* assigned backlight device */
	/* set before framebuffer registration, 
	   remove after unregister */
	struct backlight_device *bl_dev;

	/* Backlight level curve */
	struct mutex bl_curve_mutex;	
	u8 bl_curve[FB_BACKLIGHT_LEVELS];
#endif
#ifdef CONFIG_FB_DEFERRED_IO
	struct delayed_work deferred_work;
	struct fb_deferred_io *fbdefio;
#endif

	struct fb_ops *fbops;
	struct device *device; /* This is the parent */
	struct device *dev; /* This is this fb device */
	int class_flag; /* private sysfs flags */
#ifdef CONFIG_FB_TILEBLITTING
	struct fb_tile_ops *tileops; /* Tile Blitting */
#endif
	char __iomem *screen_base; /* Virtual address */
	unsigned long screen_size; /* Amount of ioremapped VRAM or 0 */ 
	void *pseudo_palette; /* Fake palette of 16 colors */ 
#define FBINFO_STATE_RUNNING	0
#define FBINFO_STATE_SUSPENDED	1
	u32 state; /* Hardware state i.e suspend */
	void *fbcon_par; /* fbcon use-only private area */
	/* From here on everything is device dependent */
	void *par;
	/* we need the PCI or similiar aperture base/size not
	   smem_start/size as smem_start may just be an object
	   allocated inside the aperture so may not actually overlap */
	resource_size_t aperture_base;
	resource_size_t aperture_size;
};

```

&emsp;&emsp;fb_info结构体中记录了关于帧缓存的全部信息，这些信息包括帧缓冲的设置参数、状态及操作函数集合。

##2.2 fb_ops结构体
&emsp;&emsp;**fb_ops结构体用来实现帧缓冲设备的操作。用户可以使用write()、read()、ioctl()函数操作设备。fb_ops结构体的定义如下：**

```c++
struct fb_ops {
	/* open/release and usage marking */
	struct module *owner;
	int (*fb_open)(struct fb_info *info, int user);
	int (*fb_release)(struct fb_info *info, int user);

	/* For framebuffers with strange non linear layouts or that do not
	 * work with normal memory mapped access
	 */
	ssize_t (*fb_read)(struct fb_info *info, char __user *buf,
			   size_t count, loff_t *ppos);
	ssize_t (*fb_write)(struct fb_info *info, const char __user *buf,
			    size_t count, loff_t *ppos);

	/* checks var and eventually tweaks it to something supported,
	 * DO NOT MODIFY PAR */
	int (*fb_check_var)(struct fb_var_screeninfo *var, struct fb_info *info);

	/* set the video mode according to info->var */
	int (*fb_set_par)(struct fb_info *info);

	/* set color register */
	int (*fb_setcolreg)(unsigned regno, unsigned red, unsigned green,
			    unsigned blue, unsigned transp, struct fb_info *info);

	/* set color registers in batch */
	int (*fb_setcmap)(struct fb_cmap *cmap, struct fb_info *info);

	/* blank display */
	int (*fb_blank)(int blank, struct fb_info *info);

	/* pan display */
	int (*fb_pan_display)(struct fb_var_screeninfo *var, struct fb_info *info);

	/* Draws a rectangle */
	void (*fb_fillrect) (struct fb_info *info, const struct fb_fillrect *rect);
	/* Copy data from area to another */
	void (*fb_copyarea) (struct fb_info *info, const struct fb_copyarea *region);
	/* Draws a image to the display */
	void (*fb_imageblit) (struct fb_info *info, const struct fb_image *image);

	/* Draws cursor */
	int (*fb_cursor) (struct fb_info *info, struct fb_cursor *cursor);

	/* Rotates the display */
	void (*fb_rotate)(struct fb_info *info, int angle);

	/* wait for blit idle, optional */
	int (*fb_sync)(struct fb_info *info);

	/* perform fb specific ioctl (optional) */
	int (*fb_ioctl)(struct fb_info *info, unsigned int cmd,
			unsigned long arg);

	/* Handle 32bit compat ioctl (optional) */
	int (*fb_compat_ioctl)(struct fb_info *info, unsigned cmd,
			unsigned long arg);

	/* perform fb specific mmap */
	int (*fb_mmap)(struct fb_info *info, struct vm_area_struct *vma);

	/* get capability given var */
	void (*fb_get_caps)(struct fb_info *info, struct fb_blit_caps *caps,
			    struct fb_var_screeninfo *var);

	/* teardown any resources to do with this framebuffer */
	void (*fb_destroy)(struct fb_info *info);
};
```

##2.3 fb_cmap结构体
&emsp;&emsp;fb_cmap结构体记录了一个颜色板信息，也可以叫调色板信息。用户空间程序可以使用ioctl()函数的FBIOGETCMAP和FBIOPITCMAP命令读取和设置颜色表的值。**fb_cmap**结构体的定义如下：

```c++
struct fb_cmap {
	__u32 start; /* First entry	*/
	__u32 len; /* Number of entries */
	__u16 *red; /* Red values	*/
	__u16 *green;
	__u16 *blue;
	__u16 *transp; /* transparency, can be NULL */
};
```
&emsp;&emsp;下面对这个结构体的各个成员进行简要的解释。

- start：表示颜色版的第一个元素入口位置。
- len：表示元素的个数。
- red：表示红色分量的值。
- green：表示绿色分量的值。
- blue：表示蓝色分量的值。
- transp：表示透明度分量的值。
&emsp;&emsp;结构体对应的存储结构体如下图所示：
 ![这里写图片描述](https://img-blog.csdn.net/20180806103422278?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NhbnRhcGFzc2VyYnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

##2.4 fb_var_screeninfo结构体
&emsp;&emsp;**fb_var_screeninfo结构体中存储了用户可以修改的显示控制器参数，例如屏幕分辨率、每个元素的比特数、透明度等。fb_var_screeninfo结构体的定义如下：**

```c++
struct fb_var_screeninfo {
	__u32 xres; /* visible resolution		*/
	__u32 yres;
	__u32 xres_virtual; /* virtual resolution		*/
	__u32 yres_virtual;
	__u32 xoffset; /* offset from virtual to visible */
	__u32 yoffset; /* resolution			*/

	__u32 bits_per_pixel; /* guess what			*/
	__u32 grayscale; /* != 0 Graylevels instead of colors */

	struct fb_bitfield red; /* bitfield in fb mem if true color, */
	struct fb_bitfield green; /* else only length is significant */
	struct fb_bitfield blue;
	struct fb_bitfield transp; /* transparency			*/	

	__u32 nonstd; /* != 0 Non standard pixel format */

	__u32 activate; /* see FB_ACTIVATE_*		*/

	__u32 height; /* height of picture in mm    */
	__u32 width; /* width of picture in mm     */

	__u32 accel_flags; /* (OBSOLETE) see fb_info.flags */

	/* Timing: All values in pixclocks, except pixclock (of course) */
	__u32 pixclock; /* pixel clock in ps (pico seconds) */
	__u32 left_margin; /* time from sync to picture	*/
	__u32 right_margin; /* time from picture to sync	*/
	__u32 upper_margin; /* time from sync to picture	*/
	__u32 lower_margin;
	__u32 hsync_len; /* length of horizontal sync	*/
	__u32 vsync_len; /* length of vertical sync	*/
	__u32 sync; /* see FB_SYNC_*		*/
	__u32 vmode; /* see FB_VMODE_*		*/
	__u32 rotate; /* angle we rotate counter clockwise */
	__u32 reserved[5]; /* Reserved for future compatibility */
};
```
&emsp;&emsp;fb_fix_screeninfo结构体中，记录了用户不能修改的固定显示控制器参数。这些固定参数如缓冲区的物理地址、缓冲区的长度、显示色彩模式、内存映射的开始位置等。这个结构体的成员都需要在驱动程序初始化时设置，该结构体的定义代码如下：

```
这里写代码片
```c++
struct fb_fix_screeninfo {
	char id[16];			/* identification string eg "TT Builtin" */
	unsigned long smem_start;	/* Start of frame buffer mem */
					/* (physical address) */
	__u32 smem_len;			/* Length of frame buffer mem */
	__u32 type;			/* see FB_TYPE_*		*/
	__u32 type_aux;			/* Interleave for interleaved Planes */
	__u32 visual;			/* see FB_VISUAL_*		*/ 
	__u16 xpanstep;			/* zero if no hardware panning  */
	__u16 ypanstep;			/* zero if no hardware panning  */
	__u16 ywrapstep;		/* zero if no hardware ywrap    */
	__u32 line_length;		/* length of a line in bytes    */
	unsigned long mmio_start;	/* Start of Memory Mapped I/O   */
					/* (physical address) */
	__u32 mmio_len;			/* Length of Memory Mapped I/O  */
	__u32 accel;			/* Indicate to driver which	*/
					/*  specific chip/card we have	*/
	__u16 reserved[3];		/* Reserved for future compatibility */
```
&emsp;&emsp;需要注意的是，**visual**表示屏幕使用的色彩模式，在Linux下，支持多种色彩模式。**line_length**表示显存中一行占用的内存字节数，具体占用多少字节由显存模式来决定。**reserved[3]**保留了6个字节为以后扩充使用。


3 LCD驱动程序分析
--

&emsp;&emsp;**LCD驱动程序以平台设备的方式实现，其中涉及关于LCD控制器的一些重要概念，下面对LCD驱动程序的主要函数进行详细的分析。**

##3.1 LCD模块的加载和卸载函数
&emsp;&emsp;**LCD设备驱动可以作为一个单独的模块加入内核，在模块的加载和卸载中需要注册和注销一个平台驱动结构体platform_driver。**
	










