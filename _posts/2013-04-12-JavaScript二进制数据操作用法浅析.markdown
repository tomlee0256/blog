---
layout: post
title:  "JavaScript 二进制数据操作用法浅析"
date:   2014-04-12 12:19:56
categories: JavaScript
---
![ta2]({{ site.url }}/assets/images/ta2.jpg)	

在JavaScript中对二进制的操作方式主要有三种：

1. Typed Array;
2. imageData;
3. charCodeAt;

这里主要浅析1的主要用法，2，3简介。

##charCodeAt
---
把原始数据当作字符串来处理，使用String.charCodeAt()方法来从数据缓冲区中读取字节。需要多次转换数据（尤其在二进制数据不是字节格式的数据时，例如32位整数或者浮点数），速度相对较慢且容易出错。

##imageData
---
imageData是在Canvas元素2D上下文环境里定义的数据类型，**imageData.data**属性是一个字节数组，大小是Canvas宽*高的四倍（每个像素有R、G、B、A四个通道）。

imageData通过**ctx.createImageData()**和**ctx.getImageData()**方法可以获取（数据处理后通过**ctx.putImageData()**方法放回到Canvas）。

##Typed Array
---
JavaScript中直接存储byte的Array类型。
不同的Typed Array类型，其每一项中可存储的byte个数不同：
<pre>
Type	           Size	 Description		                       Equivalent C Type
Int8Array	        1	  8-bit 2's complement signed integer	    signed char
Uint8Array	        1	  8-bit unsigned integer		    	    unsigned char
Uint8ClampedArray	1	  8-bit unsigned integer (clamped)	    	unsigned char
Int16Array	        2	  16-bit 2's complement signed integer	    short
Uint16Array	        2	  16-bit unsigned integer					unsigned short
Int32Array	        4	  32-bit 2's complement signed integer	    int
Uint32Array	        4	  32-bit unsigned integer					unsigned int
Float32Array	    4	  32-bit IEEE floating point				float
Float64Array	    8	  64-bit IEEE floating point				double
</pre>

##Examples
---
1.创建一个长度为1的Float32Array：

	var f32s1 = new Float32Array(1);

2.创建一个长度为4的ArrayBuffer：
//说明：ArrayBuffer长度单位为**byte**

	var b = new ArrayBuffer(4);

3.创建一个Float32Array：

	var b = new ArrayBuffer(4);
	var f32s2 = new Float32Array(b);//f32s2.length = f32s1.length;

4.create an 8-byte ArrayBuffer:

    var b = new ArrayBuffer(8);

5.create a view v1 referring to b, of type Int32, starting at the default byte index (0) and extending until the end of the buffer:
    
	var v1 = new Int32Array(b);

6.create a view v2 referring to b, of type Uint8, starting at byte index 2 and extending until the end of the buffer:

	var v2 = new Uint8Array(b, 2);

7.create a view v3 referring to b, of type Int16, starting at byte index 2 and having a length of 2:

	var v3 = new Int16Array(b, 2, 2);

description for 4~7:

![ta1]({{ site.url }}/assets/images/ta1.png)	

8.Filling each 8 consecutive floats of the new array:
{% highlight JavaScript %}

for (var i = 0; i < 128; i += 8) {
  var sub_f32s = f32s.subarray(i, i+8);
  for (var j = 0; j < 8; ++j) {
    sub_f32s[j] = j;
  }
}

{% endhighlight %}  

Note that this code uses **subarray()** to create a new Float32Array.

9.Interleaved array types Some APIs, in particular WebGL [WEBGL], can benefit from being able to use a single contiguous buffer, with interleaved data types. For example, a point might have coordinate data (3 Float32 values) followed by color data (4 Uint8 values).

For 4 points and their associated colors, this can be set up in the following way:
{% highlight JavaScript %}

var elementSize = 3 * Float32Array.BYTES_PER_ELEMENT + 4 * Uint8Array.BYTES_PER_ELEMENT;
var buffer = new ArrayBuffer(elementSize);
var coords = new Float32Array(buffer, 0);
var colors = new Uint8Array(buffer, 3 * Float32Array.BYTES_PER_ELEMENT);

{% endhighlight %}   
Note that the colors Uint8Array view is created with an explicit offset (which is given in bytes), so that the [0] element points at the 13th byte in the underlying buffer.

In this example, from any given Float32 element, to skip to the same Float32 element in the next point, 4 must be added to the index. Similarly, to skip from any given Uint8 element to the same in the next point, 16 must be added:
{% highlight JavaScript %}

var coordOffset = elementSize / Float32Array.BYTES_PER_ELEMENT;
var colorOffset = elementSize / Uint8Array.BYTES_PER_ELEMENT;

coords[0] = coords[1] = coords[2] = 1.0; // The first point's three coordinate values
colors[0] = colors[1] = colors[2] = colors[3] = 255; // The first point's four colors

coords[0+N*coordOffset] = 5.0; // The Nth point's first coordinate value
colors[0+N*colorOffset] = 128; // The Nth point's first color value

coords[i+N*coordOffset] = 6.0; // The Nth point's i coordinate value;
colors[j+N*colorOffset] = 200; // The Nth point's j color value

{% endhighlight %}   
In the above example, note that for keeping the data consistent, i must be one of 0, 1, or 2; and j must be one of 0, 1, 2, or 3. Any higher values will result in data segments that are reserved for 32-bit floats or for 8-bit integers being overwritten with incorrect data.

10.Slicing a large array into multiple regions Another usage similar to the above is allocating one large buffer, and then using different regions of it for different purposes:

	var buffer = new ArrayBuffer(1024);
      
Carve out 128 floats, 128*4 = 512 bytes in size:

	var floats = new Float32Array(buffer, 0, 128);
      
Then 128 shorts, 128*2 = 256 bytes in size, immediately following the floats. Note that the 512 byte offset argument is equal to floats.byteOffset + floats.byteLength.

	var shorts = new Uint16Array(buffer, 512, 128);
      
Finally, 256 unsigned bytes. We can write the byte offset in the form suggested above to simplify the chaining. We also let this array extend until the end of the ArrayBuffer by not explicitly specifying a length.

	var bytes = new Uint8Array(buffer, shorts.byteOffset + shorts.byteLength);
      
If the data is no longer needed, the entire 1024-byte array can be repurposed without causing additional allocations simply by creating new views and discarding the old.

##应用
---
1.Canvas图像数据与二进制格式相互转换：

* Canvas图像数据 to 二进制格式:
{% highlight JavaScript %}

imagedata = context.getImageData(0, 0, imagewidth,imageheight);
var canvaspixelarray = imagedata.data;
var canvaspixellen = canvaspixelarray.length;
var bytearray = new Uint8Array(canvaspixellen);
for (var i=0;i<canvaspixellen;++i) {
	bytearray[i] = canvaspixelarray[i];
}

{% endhighlight %}   


* 二进制格式数据 to Canvas图像数据:
{% highlight JavaScript %}

var bytearray = new Uint8Array(event.data);
var tempcanvas = document.createElement('canvas');
    tempcanvas.height= imageheight;
    tempcanvas.width= imagewidth;
var tempcontext = tempcanvas.getContext('2d');
var imgdata = tempcontext.getImageData(0,0,imagewidth,imageheight);
var imgdatalen = imgdata.data.length;
for(var i=8;i<imgdatalen;i++)
{
    imgdata.data[i] = bytearray[i];
}
tempcontext.putImageData(imgdata,0,0);

{% endhighlight %} 

2.[Web数据传输]()。

3.数据压缩，加密。

...

##常见错误
---
1.如果是用Typed Array 去分割(view) ArrayBuffer, view的range / TypedArray.BYTES_PER_ELEMENT要能**整除**。


<br/>
##更多参考
---
0.[http://www.khronos.org/registry/typedarray/specs/latest/](http://www.khronos.org/registry/typedarray/specs/latest/)

1.[http://blog.csdn.net/hfahe/article/details/7421203](http://blog.csdn.net/hfahe/article/details/7421203)

2.[http://www.adobe.com/devnet/html5/articles/real-time-data-exchange-in-html5-with-websockets.html](http://www.adobe.com/devnet/html5/articles/real-time-data-exchange-in-html5-with-websockets.html)

3.[二进制格式数据web传输 WebSocket](http://granular.cs.umu.se/browserphysics/?p=865)

4.[XMLHttpRequest2请求二进制格式数据](http://www.html5rocks.com/zh/tutorials/file/xhr2/)

##*题外话*
---
**负数**在内存中是以**补码**存储的，负数的补码：取反加1，例如：127在内存中表示为0111 1111， -127在内存中表示为~(0111 1111)+1=1000 0001。