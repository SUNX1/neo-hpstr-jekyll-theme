---
layout: post
title: JavaScript模拟GameBoy:图形
description: "Translation of \"GameBoy Emulation in JavaScript\" series articles."
modified: 2018-01-30
tags: [Translation]
categories: []
---

这是《JavaScript模拟GameBoy》系列文章的第四篇。目前已经完成了前十篇，其余的我将会继续跟进。

1. [中央处理器]({{ site.owner.site }}/2018/01/22/JavaScript模拟GameBoy-CPU.html)
2. [存储器]({{ site.owner.site }}/2018/01/25/JavaScript模拟GameBoy-存储器.html)
3. [GPU时序]({{ site.owner.site }}/2018/01/28/JavaScript模拟GameBoy-GPU时序.html)
4. [图形]({{ site.owner.site }}/2018/01/30/JavaScript模拟GameBoy-图形.html)
5. [整合]({{ site.owner.site }}/2018/02/01/JavaScript模拟GameBoy-整合.html)
6. [输入]({{ site.owner.site }}/2018/02/02/JavaScript模拟GameBoy-输入.html)
7. [精灵]({{ site.owner.site }}/2018/02/05/JavaScript模拟GameBoy-精灵.html)
8. [中断]({{ site.owner.site }}/2018/02/06/JavaScript模拟GameBoy-中断.html)
9. [内存扩展]({{ site.owner.site }}/2018/02/07/JavaScript模拟GameBoy-内存扩展.html)
10. [定时器]({{ site.owner.site }}/2018/02/24/JavaScript模拟GameBoy-Memory-定时器.html)

本系列文章的模拟器代码可以在Github查看：[http://github.com/Two9A/jsGB](http://github.com/Two9A/jsGB)

---
 
在本系列之前的几篇中，我们已经构建了GameBoy模拟器的轮廓，建立了CPU和图形处理器之间的时序，初始化了一个canvas并准备好由模拟器来绘制图形。我们已经实现了模拟GPU具体的结构，但它目前仍然无法将图形渲染到帧缓冲器。为了模拟渲染图形，必须简单介绍下GameBoy图形背后的概念。

## 背景

就像那个年代大多数的游戏机一样，GameBoy内存不够大，不能将帧缓冲直接存到内存中。取而代之的是使用图块系统（tile system），内存中保存了一组很小的位图，通过这些位图的引用来构建图（map）。这个系统的先天优势在于图可以通过图块的引用重复使用同一个图块。

GameBoy的图块图形系统使用8x8像素的图块，图中可以使用256个独立的图块。内存中可以存放两张由32x32图块组成的图，同一时刻只能显示其中一张。GameBoy内存中有384个图块的空间，所以有一半的图块是被两张图共享的：一个图使用0-255作为图块编号，而另一个使用编号-128到127作为图块编号。

在图像内存中，图块数据和图的规化如下：

| 区域 | 使用 |
| --- | --- |
| 8000-87FF | Tile set #1: tiles 0-127 |
| 8800-8FFF | Tile set #1: tiles 128-255 Tile set #0: tiles -1 to -128|
| 9000-97FF | Tile set #0: tiles 0-127 |
| 9800-9BFF | Tile map #0 |
| 9C00-9FFF | Tile map #1 |

表1: VRAM规化

定义背景时通过其图和图块数据交互来产生图形显示：

![图1：背景映射](http://imrannazar.com/content/img/jsgb-gpu-bg-map.png)
图1：背景映射
 
如前所示，背景图由32x32图块构成。总共有256x256像素。GameBoy的显示屏是160x144像素的，所以背景的范围可以相对于屏幕移动。GPU通过在背景中定义一个相对于屏幕左上角的点，通过在画面中移动这个点来使背景在屏幕上滚动。为此，GPU的寄存器Scroll X和Scroll Y存储了左上角的相对位置。

![图2：背景滚动寄存器](http://imrannazar.com/content/img/jsgb-gpu-bg-scrl.png)
图2：背景滚动寄存器

## 调色板
GameBoy通常被描述为只能显示黑色和白色的单色设备，但这并不正确，GameBoy也可以处理浅灰色和深灰色，总共四种颜色。图块数据保存这四种颜色之一需要两位，因此图块数据集中的每个图块需要8x8x2位或16字节。

GameBoy背景的另一个难题是从图块数据到最终显示之间的调色盘。图块像素的四种可能值对应四种颜色中的任何一种。这主要用于让图块集的颜色更容易变更。例如，如果保存了一组对应英文字母表的图块集，反向显示版本可以通过改变调色盘来实现，而用占用图块集的另一部分。通过改变背景调色盘GPU寄存器的值，四个调色板条目一次全部更新。

颜色的相关值和存储结构如下：

| 值 | 像素 | 模拟的颜色 |
| --- | --- | --- |
| 0 | Off | [255, 255, 255] |
| 1 | 33% on | [192, 192, 192] |
| 2 | 66% on | [96, 96, 96] |
| 3 | On | [0, 0, 0] |

表2：颜色相关值

![http://imrannazar.com/content/img/jsgb-gpu-bg-pal.png](http://imrannazar.com/content/img/jsgb-gpu-bg-pal.png)
图3:背景调色板寄存器

## 实现：图块数据
如上所述，图块数据集中的每个像素由两个比特表示，当图块在图中被引用时，这些比特被GPU读取，通过调色板然后被放到屏幕上，图块的一行数据可以同时被访问，像素通过运行位来循环访问。唯一的问题是，图块的一行是两个字节，因此其存储方案比较复杂，其中每个像素的低位保存在一个字节中，高位保存在另一个字节中。

![图4：图块数据位图结构](http://imrannazar.com/content/img/jsgb-gpu-bg-tile.png)
图4：图块数据位图结构

由于JavaScript不太适合于位图结构的快速处理，处理图块数据集时间效率最高的方式是在图像内存旁边维护一个内部数据集，并用一个更大的视图来预先计算每个像素的值。为了准确映射图块数据集，对图像内存的任何写入都必须调用更新GPU内部图块数据的函数。

    GPU.js: 内部图块数据
        _tileset: [],
    
        reset: function()
        {
            // 除了之前重置以外的代码
    	GPU._tileset = [];
    	for(var i = 0; i < 384; i++)
    	{
    	    GPU._tileset[i] = [];
    	    for(var j = 0; j < 8; j++)
    	    {
    	        GPU._tileset[i][j] = [0,0,0,0,0,0,0,0];
    	    }
    	}
        },
    
        // 将值写入VRAM然后更新内部图块数据集
        
        updatetile: function(addr, val)
        {
            // 获取这个图块行的基地址
    	addr &= 0x1FFE;
    
    	// 确定哪个图块和行已更新
    	var tile = (addr >> 4) & 511;
    	var y = (addr >> 1) & 7;
    
    	var sx;
    	for(var x = 0; x < 8; x++)
    	{
    	    // 找到这个像素的位索引
    	    sx = 1 << (7-x);
    
    	    // 更新图块集
    	    GPU._tileset[tile][y][x] =
    	        ((GPU._vram[addr] & sx)   ? 1 : 0) +
    	        ((GPU._vram[addr+1] & sx) ? 2 : 0);
        }
    }
        
    MMU.js: 图块更新触发器
        wb: function(addr, val)
        {
            switch(addr & 0xF000)
    	{
            // 只显示VRAM情况：
    	    case 0x8000:
    	    case 0x9000:
    		GPU._vram[addr & 0x1FFF] = val;
    		GPU.updatetile(addr, val);
    		break;
    	}
    }

## 实现：扫描渲染

有了这些组件，就可以开始渲染GameBoy屏幕了。由于渲染是逐行完成的，因此在渲染扫描线之前，第3部分中提到的渲染扫描函数必须找出它在屏幕上的位置。这涉及使用滚动寄存器和当前扫描线计数器计算背景图中位置的X和Y坐标。一旦确定了这个位置，扫描渲染器就能扫过图每行中的每个图块，每遇到一个图块就读入新的图块数据。

    GPU.js: 扫描渲染
        renderscan: function()
        {
        
	// 图块图的VRAM偏移量
    	var mapoffs = GPU._bgmap ? 0x1C00 : 0x1800;
    
	// Which line of tiles to use in the map
	// 使用图中哪一行图块
    	mapoffs += ((GPU._line + GPU._scy) & 255) >> 3;
    	
    	// 从一行图块中的哪个图块开始
    	var lineoffs = (GPU._scx >> 3);
    
    	// 使用图块中哪一行像素
    	var y = (GPU._line + GPU._scy) & 7;
    
    	// 从一行像素中的哪个开始
    	var x = GPU._scx & 7;
    
            // 在canvas上渲染的位置
    	var canvasoffs = GPU._line * 160 * 4;
    
    	// 从背景图中读图块索引
    	var colour;
    	var tile = GPU._vram[mapoffs + lineoffs];
    
    	// If the tile data set in use is #1, the
    	// indices are signed; calculate a real tile offset
    	if(GPU._bgtile == 1 && tile < 128) tile += 256;
    
    	for(var i = 0; i < 160; i++)
    	{

    	    // 通过调色板重新索引图块像素
    	    colour = GPU._pal[GPU._tileset[tile][y][x]];
    
    	    // 将像素绘制到canvas上
    	    GPU._scrn.data[canvasoffs+0] = colour[0];
    	    GPU._scrn.data[canvasoffs+1] = colour[1];
    	    GPU._scrn.data[canvasoffs+2] = colour[2];
    	    GPU._scrn.data[canvasoffs+3] = colour[3];
    	    canvasoffs += 4;
    
    	    // When this tile ends, read another
    	    // 如果图块结束了，读取另一个
    	    x++;
    	    if(x == 8)
    	    {
    		  x = 0;
    		  lineoffs = (lineoffs + 1) & 31;
    		  tile = GPU._vram[mapoffs + lineoffs];
    		  if(GPU._bgtile == 1 && tile < 128) tile += 256;
    	    }
    	}
    }

## 下一步: 输出
有了CPU、内存处理和图形子系统，模拟器已经几乎可以进行输出了。在第5篇中，我们将研究如何将不同的模块文件整合到一个整体的系统，加载并运行一个简单的ROM文件，将图形寄存器绑定到MMU，并设计一个简单的接口来控制模拟的运行。

> 原文链接：[GameBoy Emulation in JavaScript: Graphics](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Graphics)


