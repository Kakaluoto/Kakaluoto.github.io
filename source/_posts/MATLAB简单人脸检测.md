---
title: 【图像处理】MATLAB简单人脸检测
date: 2020-5-3
tags: [MATLAB,图像处理]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212172019558.webp
mathjax: true
---

##  1.人脸检测原理框图

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705537.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



整体思路是寻找图片中最大的连通域，将其认定为人脸。

第一个环节均值滤波，是为了减弱图像的相关细节部分，以免毛刺影响后期连通域的形成，二值化方便形态学处理，减少运算量。考虑到人脸有黑人和白人黄种人，黑人肤色较深，在二值化之后面部区域不容易形成较大的连通域，如果采取形态学边界提取的办法，就可以避免这个问题，形态学边界提取，只要结构元素够大，也可以形成较大的封闭连通域。

然后就是纵向闭合操作，这一步我选择采用竖向长条状的结构元素进行闭合运算，因为人的脸部和颈部以及头发和衣物等等都是纵向分布的，在进行形态学边界提取的时候，容易将这些靠近的成分割裂开来，这对连通域的判断极为不利，所以用竖向长条状的结构元素在在纵向进行闭合运算，将脸部上下部的区域重新连接起来。

紧接着我又用横向长条状结构元素进行横向腐蚀运算，这是因为，人的头部以下的身体部分存在有大量连通域的时候，容易对最大连通域的判决产生干扰，又因为下半部分，多半呈纵向分布，通过横向腐蚀可以将这些大块的连通域割裂开来，但是要注意的是，割裂程度不应太大，否则会使得上一步闭合操作丧失意义。

接着，由于背景杂物等因素，同样也会产生大量连通域，这会对最后结果的判决产生干扰，因此要予以剔除。

进行了层层筛选之后，在剩下的连通域里面挑一个最大的连通域，并且尺寸形状满足要求的用矩形框框起来作为人脸检测结果。

# 2 步骤

## 2.1 均值滤波

```matlab
h = ones(9)/81;
I = uint8(conv2(I,h));
figure,imshow(I),title('线性均值滤波')
```

采用9x9模板进行线性均值滤波，因为后面调用gpuArray()函数转换对输入数据有要求，所以在进行了二维卷积之后重新将数据格式转换成8位无符号整形数据。

## 2.2 二值化

```matlab
BW = imbinarize(I);
figure,imshow(BW),title('二值化')
```

直接调用imbinarize对图像进行二值化

## 2.3.形态学边界提取

```matlab
B = ones(21);%结构元素
BW = -imerode(BW,B) + BW;
figure,imshow(BW),title('形态学边界提取')
BW = bwmorph(BW,'thicken');
figure,imshow(BW),title('加粗边界')
BW = not(bwareaopen(not(BW), 300));
figure,imshow(BW),title('把空洞填了')
```

结构元素采用21x21大小的全1矩阵，先调用imrode()进行腐蚀，再用原图减去腐蚀结果，得到边界。为了让边界更加明显，调用bmworph函数，传入’thicken’参数，意在将边界加粗加厚。最后为了把空洞就是连续的小黑色块填满，调用bwareaopen()函数，该函数是去除面积小于300的白块，为了去除黑块，先调用not(BW)给原来的二值图像取反，去掉取反后面积小于300的白块，再次取反，达到去掉面积小于300的黑块的目的。

## 2.4 纵向闭合与横向腐蚀

```matlab
%进行形态学运算
B = strel('line',50,90);
BW = imdilate(BW,B);
BW = imerode(BW,B);
figure,imshow(BW),title('再闭操作之后')
B = strel('line',10,0);
BW = imerode(BW,B);
figure,imshow(BW),title('闭操作之后再腐蚀')
BW = gpuArray(BW);
```

调用strel()函数生成特定的结构元素，第一步调用strel(‘line’,50,90)，意思是调用直线型结构元素，长度为50，角度90度，也就是竖直的长条形结构元素。接着利用B调用imdilate和imerode，先进行膨胀再进行腐蚀完成闭操作运算。下一步，继续生成横向长条形结构元素，进行腐蚀操作，注意这里的结构元素不宜面积太大，长度太长，否则会过度影响上一步的结果

最后为了循环过程中提升运算速度，将数据类型更改为gpuArray()，可以在GPU上进行计算，节省时间。

## 2.5 消除边界多余连通域

```matlab
%最小化背景
%细分
div = 10;
r = floor(n1/div);%分成10块 行
c = floor(n2/div);%分成10块 列
x1 = 1;x2 = r;%对应行初始化
s = r*c;%块面积
%判断人脸是否处于图片四周，如果不是就全部弄黑
%figure
for i=1:div
    y1 = 1;y2 = c;%对应列初始化
    for j=1:div
        loc = find(BW(x1:x2,y1:y2)==0);%统计这一块黑色像素的位置
        num = length(loc);
        rate = num*100/s;%统计黑色像素占比
        if (y2<=0.2*div*c||y2>=0.8*div*c)||(x1<=r||x2>=r*div)
            if rate <=100
                BW(x1:x2,y1:y2) = 0;
            end
            %imshow(BW)
        else
            if rate <=25
                BW(x1:x2,y1:y2) = 1;
            end
            %imshow(BW)
        end%下一列
        y1 = y1 + c;
        y2 = y2 + c;
    end%下一行
    x1 = x1 + r;
    x2 = x2 + r;
end
```

对于周围多余的杂物产生的连通域，我选择先将整幅图像划分为很多小块，对一部分在图像边缘的小块全部置成黑色，对不在边缘的通过计算其中黑色像素比例来判定是否应该全部给成白色。因为图片中央可能还会存在一些细小空洞，这些空洞在每一小块占比不是很大但有可能影响连通性，如果一个小块里面大部分是白色就全部给成白色，方便后期判定最大连通域。

Div是均分比例，我这里设置成10，也就是将整个图像划分成100个小块。r就是在行方向上一个小块占多少像素，c就是在列方向上一个小块有多少像素。用两层循环来遍历每个小块，通过find(x1:x2,y1:y2)==0返回所有满足要求的像素的索引，索引长度就是黑色像素的个数。

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705088.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

通过条件判断是否在边界。

##   2.6 寻找最大连通域并画框

```matlab
figure
subplot(1,2,1)
imshow(BW)
title('最终处理')
L = bwlabel(BW,8);%利用belabel函数对8连通域区间进行标号
BB = regionprops(L,'BoundingBox');%得到矩形框，框柱每一个连通域
BB = cell2mat(struct2cell(BB));
[s1,s2] = size(BB);
BB = reshape(BB,4,s1*s2/4)';
pickshape = BB(:,3)./BB(:,4);%
shapeind = BB(0.3<pickshape&pickshape<3,:);%筛选掉尺寸比例不合格
[~,arealind] = max(shapeind(:,3).*shapeind(:,4));
subplot(1,2,2)
imshow(rgb)
hold on
rectangle('Position',[shapeind(arealind,1),shapeind(arealind,2),shapeind(arealind,3),shapeind(arealind,3)],...
    'EdgeColor','g','Linewidth',2)
title('人脸检测')
```

经过消除边界多余连通域后得到的BW中剩下的连通域内存在着我们需要的那个对应人脸的连通域。一般情况下，对应人脸的那个连通域会是最大的连通域。

调用bwlabel(BW,8)函数给所有连通域标记，其中邻域规则采用8邻域，返回一个被标记过连通域的图像，第k个被标记的连通域所有像素值为k。

调用regionprops(L,’BoundingBox’)函数，传入参数L和’BoundingBox’，该函数用于获取图像的各种属性，传入’BoundingBox’返回的是一个结构体，每一个结构体内都包含了一个能框柱其对应连通域的最小方框。一个方框，用一个序列来描述[x,y,width,height]，这个序列包含了方框左上角像素的坐标以及长和宽。

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705913.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

这三条语句，第一条用来将结构体转换成矩阵向量，方便计算。第二行获取这个矩阵的维度。由于BB刚转换成向量的时候，是一个行向量，每4个元素1组对应一个方框。为了后续计算方便，使用reshape()函数，将BB重构成一个矩阵，这个矩阵有4列，每一列对应方框的一个参数，比如坐标，长宽等等。每一行对应一个方框。

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705141.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

第一行计算长宽比，得到的pickshape向量的每一行对应每个方框的长宽比。由于有的方框明显过于扁平或者过于狭长，这种方框应该是要扔掉的。

所以第二行，通过逻辑表达式从BB内筛选出尺寸比例合格的方框，存在shapeind里面。
 剩下的尺寸符合要求的方框里面要选出面积最大的那个，最后一行，得到面积最大的方框对应的索引。

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705367.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

把方框画出来。

#   3 检测结果

![img](https://s4.ax1x.com/2022/01/12/7npWH1.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705681.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

图 15rgb图像转换成灰度图像         图 16线性均值滤波结果

可以看到，均值滤波使得图像变模糊了细节减少

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705473.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

图 17二值化结果                   图 18形态学边界提取结果

以看到边界被成功提取了出来，在人脸部形成了一个比较大的连通域

![img](https://s4.ax1x.com/2022/01/12/7npqud.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)


 可以看到，进行边界加粗和空洞添补之后，眼睛部分的黑块被消除了，这使得脸部连通域更大了

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171705658.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

注意观察图片左边的相分离的白块，在纵向闭操作之后连在了一起，同时脸部连通域进一步扩大，然后横向腐蚀在尽量维持脸部连通域大小的情况下减小了图片下方连通域。

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706690.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



可以看到效果还可以。还有其他的测试结果

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706766.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



图 24测试样例

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706233.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



图 25测试样例

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706396.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



图 26测试样例

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706231.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



图 27测试样例

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706654.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



图 28测试样例

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706167.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



图 29测试样例

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706730.png)![点击并拖拽以移动](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171706730.png)



图 30测试样例

这里注意到，黑人同样也被检测到了

```matlab
%%%完整代码
rgb = imread('f.jpg');
I = rgb2gray(rgb);
[n1,n2] = size(I);
%灰度图
figure,imshow(I),title('灰度图')
tic
h = ones(9)/81;
I = uint8(conv2(I,h));
figure,imshow(I),title('线性均值滤波')
BW = imbinarize(I);
figure,imshow(BW),title('二值化')
B = ones(21);%结构元素
BW = -imerode(BW,B) + BW;
figure,imshow(BW),title('形态学边界提取')
BW = bwmorph(BW,'thicken');
figure,imshow(BW),title('加粗边界')
BW = not(bwareaopen(not(BW), 300));
figure,imshow(BW),title('把空洞填了')
%进行形态学运算
B = strel('line',50,90);
BW = imdilate(BW,B);
BW = imerode(BW,B);
figure,imshow(BW),title('再闭操作之后')
B = strel('line',10,0);
BW = imerode(BW,B);
figure,imshow(BW),title('闭操作之后再腐蚀')
BW = gpuArray(BW);

%最小化背景
%细分
div = 10;
r = floor(n1/div);%分成10块 行
c = floor(n2/div);%分成10块 列
x1 = 1;x2 = r;%对应行初始化
s = r*c;%块面积
%判断人脸是否处于图片四周，如果不是就全部弄黑
%figure
for i=1:div
    y1 = 1;y2 = c;%对应列初始化
    for j=1:div
        loc = find(BW(x1:x2,y1:y2)==0);%统计这一块黑色像素的位置
        num = length(loc);
        rate = num*100/s;%统计黑色像素占比
        if (y2<=0.2*div*c||y2>=0.8*div*c)||(x1<=r||x2>=r*div)
            if rate <=100
                BW(x1:x2,y1:y2) = 0;
            end
            %imshow(BW)
        else
            if rate <=25
                BW(x1:x2,y1:y2) = 1;
            end
            %imshow(BW)
        end%下一列
        y1 = y1 + c;
        y2 = y2 + c;
    end%下一行
    x1 = x1 + r;
    x2 = x2 + r;
end

figure
subplot(1,2,1)
imshow(BW)
title('最终处理')
L = bwlabel(BW,8);%利用belabel函数对8连通域区间进行标号
BB = regionprops(L,'BoundingBox');%得到矩形框，框柱每一个连通域
BB = cell2mat(struct2cell(BB));
[s1,s2] = size(BB);
BB = reshape(BB,4,s1*s2/4)';
pickshape = BB(:,3)./BB(:,4);%
shapeind = BB(0.3<pickshape&pickshape<3,:);%筛选掉尺寸比例不合格
[~,arealind] = max(shapeind(:,3).*shapeind(:,4));
subplot(1,2,2)
imshow(rgb)
hold on
rectangle('Position',[shapeind(arealind,1),shapeind(arealind,2),shapeind(arealind,3),shapeind(arealind,3)],...
    'EdgeColor','g','Linewidth',2)
title('人脸检测')
toc
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
