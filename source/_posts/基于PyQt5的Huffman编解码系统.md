---
title: 【数据结构】基于PyQt5的Huffman编解码系统
date: 2020-7-4
tags: [数据结构,Qt,Python]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171714769.png
mathjax: true
---

 最近布置的作业要整个编码压缩译码解压缩系统，唉真是难啊，临时花了几天看PyQt5，踩了无数坑，终于算是弄完了。

弄了动态背景，其实就是GIF图播放，计算任务是放在子线程里面的，任务进度在主线程UI界面更新。

参考博客：

Huffman编码：

[基于哈夫曼编码的压缩算法的Python实现_字节莫的CSDN博客-CSDN博客_python 哈夫曼压缩](https://blog.csdn.net/weixin_43690347/article/details/84146979)

[Python中使用哈夫曼算法实现文件的压缩与解压缩_cc815107613的博客-CSDN博客_python基于哈夫曼编码的文件压缩及解压缩](https://blog.csdn.net/cc815107613/article/details/103260408)

Pyqt5多线程：

[pyqt5 的多线程（QThread）遇到的坑（一）_HHKJ 的博客-CSDN博客](https://blog.csdn.net/wuwei_201/article/details/104720386)

[pyqt5 的多线程（QThread）遇到的坑（二）_HHKJ 的博客-CSDN博客_pyqt多线程闪退](https://blog.csdn.net/wuwei_201/article/details/104803019)

Pyqt5刷新页面：

[pyqt5学习笔记——刷新页面_OneKey-CSDN博客_pyqt5 刷新界面](https://blog.csdn.net/qq_34765552/article/details/78860540)

Pyqt5打包：

[[PyQt\] 使用.qrc 生成资源文件供程序中使用_的专栏-CSDN博客_pyqt 编译qrc文件](https://blog.csdn.net/wn0112/article/details/47973953)

[PyQt5，资源文件 .qrc 的使用_龚建波-CSDN博客_pyqt5 qrc](https://blog.csdn.net/gongjianbo1992/article/details/105361880)

先上效果图：

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171715223.gif)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171715348.gif)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

压缩解压缩结果分析：**结果与分析**

测试用样本

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171714057.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 压缩结果

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171714637.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

不同格式压缩率统计

| 文件格式  | .mp3   | .wav   | .bmp  | .jpg | .gif   | .doc | .txt   | .py    |
| --------- | ------ | ------ | ----- | ---- | ------ | ---- | ------ | ------ |
| 压缩前/KB | 10012  | 244    | 2026  | 227  | 1980   | 20   | 330    | 21     |
| 压缩后/KB | 9998   | 205    | 1001  | 227  | 1965   | 14   | 241    | 14     |
| 压缩率    | 0.9986 | 0.8401 | 0.494 | 1    | 0.9924 | 0.7  | 0.7303 | 0.6667 |

通过数据可以看到，文件多少有被压缩过；对于原始数据格式像bmp图片格式和wav音频格式这类格式，或者txt文本，压缩率是比较低的，效果较好。像mp3和jpg这类已经被高度压缩过的文件格式，数据冗余较少，可供压缩的空间不多。其中bmp格式图片压缩率明显低于其它，还有一个可能的原因是，该图像虽然体积大，但是属于像素风图像，颜色阶数较少，像素感很明显。

增大数据样本量进行统计

 平均压缩率

| 文件格式   | .mp3   | .wav   | .bmp   | .jpg | .txt   |
| ---------- | ------ | ------ | ------ | ---- | ------ |
| 平均压缩率 | 0.9986 | 0.8173 | 0.7768 | 1    | 0.7238 |



但是，因为采用的是Huffman算法，压缩率与输入符号的概率分布有关，越是均匀分布的数据，冗余越少，压缩率越高，压缩效果自然不是很好，反而越是分布集中甚至单一的数据存在大量的数据冗余，压缩率理论上会比较低，压缩效果会更好。于是我选取了一些比较典型的图片，来验证我的想法。



![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171715817.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
 图片格式统一采用bmp格式，两张颜色分布均匀色彩丰富的图和两张颜色分布集中的图片，来看一下效果。

图 16压缩结果

![img](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171715130.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

压缩结果统计

|           | 色彩丰富1.bmp | 色彩丰富2.bmp | 色彩集中1.bmp | 色彩集中2.bmp |
| --------- | ------------- | ------------- | ------------- | ------------- |
| 压缩前/KB | 5401          | 2002          | 2453          | 6751          |
| 压缩后/KB | 5376          | 1964          | 2360          | 6425          |
| 压缩率    | 0.9953        | 0.981         | 0.962         | 0.9517        |

可以看到，色彩丰富，数据分布较为均匀的图像压缩率高，压缩效果不好，色彩分布集中较为单一的图像，压缩率低，压缩效果好。此外，色彩集中2.bmp颜色比色彩集中1.bmp颜色更为单调，对应压缩率也更低，压缩效果更好。

代码由于注释比较多，就不啰嗦了。

Huffman编码部分：主要是参考大佬们的，最后用类封装了一下

```python
import sys
from PyQt5.QtCore import QThread, pyqtSignal, QObject

sys.setrecursionlimit(1000000)  # 压缩大文件实时会出现超出递归深度，故修改限制
import time


# 节点类定义
class node(object):
    def __init__(self, value=None, left=None, right=None, father=None):
        self.value = value  # 节点的权值
        self.left = left  # 左节点
        self.right = right  # 右节点
        self.father = father  # 父节点

    def build_father(left, right):  # 构造父节点
        n = node(value=left.value + right.value, left=left, right=right)  # 子节点权值相加
        left.father = right.father = n
        return n

    def encode(n):  # huffman编码，从下往上递归遍历
        if n.father == None:
            return b''
        if n.father.left == n:
            return node.encode(n.father) + b'0'  # 左节点编码0
        else:
            return node.encode(n.father) + b'1'  # 右节点编码为1


# 只有继承了QObject类才可以使用信号
class HuffmanEncoder(QObject, object):
    progress = pyqtSignal(int)  # 发送进度信号，这个类可以向外发送当前进度值

    def __init__(self, node_dict=None, count_dict=None, ec_dict=None, nodes=None, inverse_dict=None):
        super(HuffmanEncoder, self).__init__()
        if node_dict is None:
            node_dict = {}
            # 存储节点的字典，key为读入的字节，value为对应的节点对象
        if count_dict is None:
            count_dict = {}
            # 字符频率对应字典，key为读入的字节(字符)，value为该字节出现次数，为了解码重构哈夫曼树
        if ec_dict is None:
            ec_dict = {}
            # 符号编码表 key:字节(符号),value:编码如b'1001000'，都是字符串
        if nodes is None:
            nodes = []
            # 存放节点的列表
        if inverse_dict is None:
            inverse_dict = {}
            # 反向字典，key:编码 value:编码对应的字符
        self.node_dict = node_dict
        self.count_dict = count_dict
        self.ec_dict = ec_dict
        self.nodes = nodes
        self.inverse_dict = inverse_dict
        self.temp = 0  # 当前进度，用于向外发送信号

    # 构造哈夫曼树
    def build_tree(self, l):
        if len(l) == 1:
            return l
            # 节点列表只剩一个根节点的时候，返回
            # 此时根节点连接了两个子节点，子节点又连接了孙节点，可以通过叶子节点递归遍历
        sorts = sorted(l, key=lambda x: x.value, reverse=False)  # 根据节点的权值进行排序
        n = node.build_father(sorts[0], sorts[1])  # 权值最小的两个节点，生成父节点
        sorts.pop(0)  # 将节点列表里面节点权值最小的丢掉
        sorts.pop(0)  # 继续把参与合并的第二个节点丢掉
        sorts.append(n)  # 把合并之后得到新权值的父节点，加入节点列表
        return self.build_tree(sorts)  # 递归构造

    # 可以看出，因为每次都是选择最小的两个节点，其中较小的那个节点做左节点，较大的做右节点
    # 所以编码结果是唯一的，与手工编码随机选取左右节点不同

    # 当树构建好之后调用，根据每个叶子结点，从下往上编码
    def encode(self, echo):
        # node_dict存储节点的字典，key为读入的字节，value为对应的节点对象
        for x in self.node_dict.keys():
            # ec_dict[x]符号编码表 key:字节(符号),value:编码如b'1001000'
            self.ec_dict[x] = node.encode(self.node_dict[x])
            if echo:  # 输出编码表（用于调试）
                print(x)
                print(self.ec_dict[x])

    # 编码函数
    def encodefile(self, inputfile, outputfile):
        node_dict = self.node_dict
        # node_dict存储节点的字典，key为读入的字节，value为对应的节点对象
        count_dict = self.count_dict
        # 字符频率对应字典，key为读入的字节(字符)，value为该字节出现次数，为了解码重构哈夫曼树
        ec_dict = self.ec_dict
        # ec_dict[x]符号编码表 key:字节(符号),value:编码如b'1001000'
        print("Starting encode...")
        f = open(inputfile, "rb")
        bytes_width = 1  # 每次读取的字节宽度
        i = 0

        f.seek(0, 2)
        count = f.tell() / bytes_width  # 一共有多少个符号数
        print(count)
        nodes = []  # 结点列表，用于构建哈夫曼树
        buff = [b''] * int(count)  # 初始化字节存储列表buff
        f.seek(0)

        # 计算字符频率,并将单个字符构建成单一节点
        while i < count:
            buff[i] = f.read(bytes_width)  # 每次读取bytes_width个字节
            if count_dict.get(buff[i], -1) == -1:
                count_dict[buff[i]] = 0  # key:buff[i] ，value:0
            count_dict[buff[i]] = count_dict[buff[i]] + 1
            i = i + 1
        print("Read OK")
        print(count_dict)  # 输出权值字典,可注释掉
        for x in count_dict.keys():
            node_dict[x] = node(count_dict[x])
            # 生成一个频率为count_dict[x]的节点，存入字典 node_dict[x]
            nodes.append(node_dict[x])
            # 把这个节点加入节点列表

        f.close()
        tree = self.build_tree(nodes)  # 哈夫曼树构建
        self.encode(False)  # 构建编码表
        print("Encode OK")
        # sorted_nodes是被排过序的节点列表[(key1,value1),(key2,value2)...]
        # 每个元素是一个元组(key,value)，其中key是对应的字符(字节),value是该字符出现的频率
        sorted_nodes = sorted(count_dict.items(), key=lambda x: x[1], reverse=True)
        # 对所有根节点进行排序，找出频率最高的节点
        bit_width = 1
        print("head:", sorted_nodes[0][1])
        # 动态调整编码表的字节长度，优化文件头大小，sorted_nodes[0][1]即value1，最大的频率值
        # 计算存储最大频率值需要的字节数
        if sorted_nodes[0][1] > 255:
            bit_width = 2
            if sorted_nodes[0][1] > 65535:
                bit_width = 3
                if sorted_nodes[0][1] > 16777215:
                    bit_width = 4
        print("bit_width:", bit_width)
        i = 0  # 计数变量，用于遍历所有字节
        byte_written = 0b1
        # 初始化为1占位，移位运算调用bit_length判断当前长度，这个变量是要被写入硬盘的

        o = open(outputfile, 'wb')
        name = inputfile.split('/')
        o.write((name[len(name) - 1] + '\n').encode(encoding="utf-8"))  # 写出原文件名
        o.write(int.to_bytes(len(ec_dict), 2, byteorder='big'))  
        # 写出不同符号种类数，即叶子结点总数
        o.write(int.to_bytes(bit_width, 1, byteorder='big'))  # 写出编码表字节宽度
        for x in ec_dict.keys():  # 编码文件头
            o.write(x)  # 写入符号
            o.write(int.to_bytes(count_dict[x], bit_width, byteorder='big')) 
            # 写入符号对应频率

        print('head OK')
        # 注意是按字节写入
        while i < count:  # 开始压缩数据,一个一个字节遍历，将编码结果写入
            for x in ec_dict[buff[i]]:
                # buff[i]是一个符号(字节)
           #作为key从编码字典ec_dict[buff[i]]取出一个编码b'1100...111000..'，类型是字符串
                byte_written = byte_written << 1  # 右移腾出空位
                if x == 49:  # 如果，x当前是'1'，那就将byte_written最后一位置1
                    byte_written = byte_written | 1
                if byte_written.bit_length() == 9:
                    # 一个字节有8位，9位包含了第一位是1的那个占位符
                    #因为bit_length只从第一个非0位算起
                    byte_written = byte_written & (~(1 << 8))  # 取出一个字节，即低8位
                    o.write(int.to_bytes(byte_written, 1, byteorder='big'))
                    o.flush()  # 立即写入，更新缓冲区
                    byte_written = 0b1  # 置1复位
            tem = int(i / len(buff) * 100)
            if tem > 0:
                if tem - self.temp >= 1:  # 防止频繁发送信号阻塞主线程UI
                    print("encode:", tem, '%')  # 输出压缩进度
                    if tem > 95:
                        self.temp = 100
                    else:
                        self.temp = tem
                    self.progress.emit(self.temp)  # 发送当前进度
            i = i + 1

        if byte_written.bit_length() > 1:  # 处理文件尾部不足一个字节的数据
            byte_written = byte_written << (8 - (byte_written.bit_length() - 1))
            byte_written = byte_written & (~(1 << byte_written.bit_length() - 1))
            o.write(int.to_bytes(byte_written, 1, byteorder='big'))
        o.close()
        self.node_dict = node_dict
        self.count_dict = count_dict
        self.ec_dict = ec_dict
        self.nodes = nodes
        print("File encode successful.")

    def decodefile(self, inputfile, outputfile):
        node_dict = self.node_dict
        # node_dict存储节点的字典，key为读入的字节，value为对应的节点对象
        ec_dict = self.ec_dict
        # 字符频率对应字典，key为读入的字节(字符)，value为该字节出现次数，为了解码重构哈夫曼树
        inverse_dict = self.inverse_dict
        # 反向字典，key:编码 value:编码对应的字符
        nodes = self.nodes  # 存放节点的列表
        print("Starting decode...")
        count = 0
        byte_written = 0
        f = open(inputfile, 'rb')
        f.seek(0, 2)
        eof = f.tell()  # 获取文件末尾位置
        f.seek(0)
        outputfile = (outputfile + f.readline().decode(encoding="utf-8")).replace('\n', '')
        # 文件保存路径和文件名结合生成文件指针
        o = open(outputfile, 'wb')
        count = int.from_bytes(f.read(2), byteorder='big')  
        # 取出叶子结点数量，也就是不同符号种类数
        bit_width = int.from_bytes(f.read(1), byteorder='big')  # 取出编码表字宽
        i = 0
        de_dict = {}
        while i < count:  # 解析文件头
            key = f.read(1)  # 取出符号
            value = int.from_bytes(f.read(bit_width), byteorder='big')  # 取出符号对应频率
            de_dict[key] = value  # 建立符号频率表 key:符号 value:该符号出现次数
            i = i + 1
        for x in de_dict.keys():
            node_dict[x] = node(de_dict[x])
            nodes.append(node_dict[x])
        tree = self.build_tree(nodes)  # 重建哈夫曼树
        self.encode(False)  # 建立编码表，此时产生 self.ec_dict编码字典，key:符号，value:b'010101....1010..'
        for x in ec_dict.keys():  # 反向字典构建
            inverse_dict[ec_dict[x]] = x  # key和value对调,key:是编码b'010101....1010..',value:是x即符号，8位
        i = f.tell()  # 获取当前指针位置
        data = b''
        while i < eof:  # 开始解压数据
            byte_written = int.from_bytes(f.read(1), byteorder='big')
            # print("byte_written:",byte_written)
            i = i + 1
            j = 8  # 一个字节八位
            while j > 0:
                if (byte_written >> (j - 1)) & 1 == 1:  # 取最高位判断
                    data = data + b'1'
                    byte_written = byte_written & (~(1 << (j - 1))) 
                    # 去掉最高位，保留剩下几位
                else:
                    data = data + b'0'
                    byte_written = byte_written & (~(1 << (j - 1)))
                if inverse_dict.get(data, 0) != 0: 
                    # key:是编码b'010101....1010..',value:是x即符号，8位
                    o.write(inverse_dict[data])
                    o.flush()
                    # print("decode",data,":",inverse_dict[data])
                    data = b''  
             # 如果匹配到了就清零，如果没有就不清零，比如码长大于8，不会清零会继续变长直到匹配
                j = j - 1
            tem = int(i / eof * 100)
            if tem > 0:
                if tem - self.temp >= 1:
                    print("decode:", tem, '%')  # 输出解压进度
                    if tem > 95:
                        self.temp = 100
                    else:
                        self.temp = tem
                    self.progress.emit(self.temp)
            byte_written = 0

        f.close()
        o.close()
        print("File decode successful.")
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

UI部分：这里比较繁琐，不需要的不要也行

```python
from PyQt5 import QtCore, QtGui, QtWidgets
from PyQt5.QtCore import (Qt, pyqtSignal, pyqtSlot, QThread, QTimer)
from PyQt5.QtWidgets import QWidget
from PyQt5.QtGui import (QMovie, QIcon, QCursor)
from PyQt5.QtWidgets import (QMainWindow, QApplication, QVBoxLayout, QLabel,
                             QTabWidget, QPushButton, QHBoxLayout, QPlainTextEdit,
                             QFileDialog, QDesktopWidget, QDialog, QProgressBar)
import images_rc  # 加载图片资源文件
import sys
import os
import time
import huffman
from huffman import HuffmanEncoder


class Ui_MainWindow(QTabWidget):
    def __init__(self, parent=None):
        super(Ui_MainWindow, self).__init__(parent)
        self.setupUi()

    def setupUi(self):
        self.setGeometry(0, 0, 550, 550)
        self.setFixedSize(550, 550)
        self.tab1 = QWidget()
        self.tab2 = QWidget()
        self.addTab(self.tab1, '压缩')
        self.addTab(self.tab2, '解压')
        self.setWindowTitle('Huffman压缩-解压')
        self.setWindowIcon(QIcon(':/image/bird.ico'))
        self.loadpath = ''  # 导入文件路径
        # 控件样式定义
        self.style = '''
                    #btn_code{
                        border-radius: 4px;
                        border-image: url(':/image/code.png');
                        }
                    #btn_code:Pressed{
                        border-image: url(':/image/code_press.png');
                        }
    
                    #btn_select_file1{
                        border-radius: 4px;
                        border-image: url(':/image/sele_file.png');
                        }
                    #btn_select_file1:Pressed{
                        border-image: url(':/image/sele_file_press.png');
                        }
                    #btn_select_file2{
                        border-radius: 4px;
                        border-image: url(':/image/sele_file.png');
                        }
                    #btn_select_file2:Pressed{
                        border-image: url(':/image/sele_file_press.png');
                        }
                    #btn_decode{
                                border-radius: 4px;
                                border-image: url(':/image/decode.png');
                        }
                    #btn_decode:Pressed{
                        border-image: url(':/image/decode_press.png');
                        }  
                    #dir_edit1{
                        background-color: rgba(255, 255, 255, 0);
                        font-family:微软雅黑;
                        font-size:14px
                        }
                    #dir_edit2{
                        background-color: rgba(255, 255, 255, 0);
                        font-family:微软雅黑;
                        font-size:14px
                        }
                    #btn_change_dir1{
                        border-radius: 4px;
                        border-image: url(':/image/change_dir.png');
                        }
                    #btn_change_dir1:Pressed{
                        border-image: url(':/image/change_dir_press.png');
                        }
                    #btn_change_dir2{
                        border-radius: 4px;
                        border-image: url(':/image/change_dir.png');
                        }
                    #btn_change_dir2:Pressed{
                        border-image: url(':/image/change_dir_press.png');
                        }
                '''
        self.tab1UI()  # 压缩标签页初始化
        self.tab2UI()  # 解压标签页初始化
        self.center()

    def center(self):  # 窗口居中函数
        screen = QDesktopWidget().screenGeometry()
        size = self.geometry()
        self.move((screen.width() - size.width()) / 2,
                  (screen.height() - size.height()) / 2)

    def tab1UI(self):
        # 动态背景
        self.gif = QMovie(':/image/bgi.gif')
        label = QLabel(self.tab1)
        label.setMovie(self.gif)
        label.setFixedSize(550, 550)
        label.setContentsMargins(0, 0, 0, 0)
        # 标题
        label_title = QLabel(self.tab1)
        label_title.setGeometry(0, 0, 400, 50)
        label_title.setPixmap(QtGui.QPixmap(':/image/title.png'))
        label_title.setScaledContents(True)
        # 开始压缩按钮
        btn_code = QPushButton(self.tab1)
        btn_code.setGeometry(350, 420, 170, 55)
        btn_code.setObjectName('btn_code')
        btn_code.setStyleSheet(self.style)
        btn_code.setCursor(QCursor(Qt.PointingHandCursor))
        btn_code.clicked.connect(self.start_coding)  # 连接槽函数

        # 操作区域布局
        menu_layout = QVBoxLayout()
        # 选择文件按钮
        select_file_layout = QHBoxLayout()
        btn_select_file1 = QPushButton()
        btn_select_file1.setObjectName('btn_select_file1')
        btn_select_file1.setGeometry(0, 0, 120, 35)
        btn_select_file1.setFixedSize(120, 35)
        btn_select_file1.setStyleSheet(self.style)
        btn_select_file1.setCursor(QCursor(Qt.PointingHandCursor))
        btn_select_file1.clicked.connect(self.select_file)

        label_filename = QLabel('filename')
        label_filename.setObjectName('label_filename')
        label_filename.setGeometry(0, 0, 120, 35)
        label_filename.setFixedHeight(35)
        select_file_layout.addWidget(btn_select_file1)
        select_file_layout.addWidget(label_filename, Qt.AlignRight)
        select_file_layout.setContentsMargins(0, 0, 0, 20)
        self.label_filename1 = label_filename  # 为了回调函数调用存在类变量里面
        # 载入文件子布局
        load_file_layout = QHBoxLayout()
        # “压缩到“子控件
        dir_to_code = QLabel()
        dir_to_code.setFixedSize(95, 35)
        dir_to_code.setPixmap(QtGui.QPixmap(':/image/code_to_dir.png'))
        dir_to_code.setScaledContents(True)
        # ”输入路径“子控件
        dir_edit1 = QPlainTextEdit()
        dir_edit1.setObjectName('dir_edit1')
        dir_edit1.setFixedHeight(35)
        dir_edit1.setObjectName('dir_edit1')
        dir_edit1.setStyleSheet(self.style)
        self.dir_edit1 = dir_edit1
        # "更改路径"子控件
        btn_change_dir1 = QPushButton()
        btn_change_dir1.setObjectName('btn_change_dir1')
        btn_change_dir1.setFixedSize(120, 35)
        btn_change_dir1.setStyleSheet(self.style)
        btn_change_dir1.setCursor(QCursor(Qt.PointingHandCursor))
        btn_change_dir1.clicked.connect(self.change_directory)
        self.btn_change_dir1 = btn_change_dir1
        # 子控件加入子布局(水平布局)
        load_file_layout.addWidget(dir_to_code)
        load_file_layout.addWidget(dir_edit1)
        load_file_layout.addWidget(btn_change_dir1)
        load_file_layout.setContentsMargins(0, 0, 0, 20)
        # 选择文件按钮布局
        choose_file = QWidget()
        choose_file.setGeometry(0, 35, 200, 35)
        choose_file.setFixedSize(200, 35)
        choose_file.setLayout(select_file_layout)
        # 子布局和load_file子控件结合
        load_file = QWidget()
        load_file.setGeometry(0, 35, 540, 35)
        load_file.setFixedSize(540, 35)
        load_file.setLayout(load_file_layout)
        # 子控件载入布局
        menu_layout.addWidget(choose_file)
        menu_layout.addWidget(load_file)
        # 纵向控件，包含选择文件按钮和路径输入框等
        menu = QWidget(self.tab1)
        menu.setGeometry(-10, 50, 540, 100)
        menu.setFixedSize(540, 100)
        menu.setLayout(menu_layout)
        self.gif.start()

    def tab2UI(self):
        # 动态背景
        self.gif = QMovie(':/image/bgi3.gif')
        label = QLabel(self.tab2)
        label.setMovie(self.gif)
        label.setFixedSize(550, 550)
        label.setContentsMargins(0, 0, 0, 0)
        # 标题
        label_title = QLabel(self.tab2)
        label_title.setGeometry(0, 0, 400, 50)
        label_title.setPixmap(QtGui.QPixmap(':/image/title2.png'))
        label_title.setScaledContents(True)
        # 开始解压缩按钮
        btn_decode = QPushButton(self.tab2)
        btn_decode.setGeometry(350, 420, 170, 55)
        btn_decode.setObjectName('btn_decode')
        btn_decode.setStyleSheet(self.style)
        btn_decode.setCursor(QCursor(Qt.PointingHandCursor))
        btn_decode.clicked.connect(self.start_decoding)  # 解压按钮的信号连接到解压按钮的槽函数
        # 操作区域布局
        menu_layout = QVBoxLayout()
        # 选择文件按钮
        select_file_layout = QHBoxLayout()
        btn_select_file2 = QPushButton()
        btn_select_file2.setObjectName('btn_select_file2')
        btn_select_file2.setGeometry(0, 0, 120, 35)
        btn_select_file2.setFixedSize(120, 35)
        btn_select_file2.setStyleSheet(self.style)
        btn_select_file2.setCursor(QCursor(Qt.PointingHandCursor))
        btn_select_file2.clicked.connect(self.select_file)  # 选择文件信号连接到选择文件槽函数

        label_filename = QLabel('filename')
        label_filename.setObjectName('label_filename')
        label_filename.setGeometry(0, 0, 120, 35)
        label_filename.setFixedHeight(35)
        select_file_layout.addWidget(btn_select_file2)
        select_file_layout.addWidget(label_filename, Qt.AlignRight)
        select_file_layout.setContentsMargins(0, 0, 0, 20)
        self.label_filename2 = label_filename
        # 载入文件子布局
        load_file_layout = QHBoxLayout()
        # “解压缩到“子控件
        dir_to_decode = QLabel()
        dir_to_decode.setFixedSize(95, 35)
        dir_to_decode.setPixmap(QtGui.QPixmap(':/image/decode_to_dir.png'))
        dir_to_decode.setScaledContents(True)
        # ”输入路径“子控件
        dir_edit2 = QPlainTextEdit()
        dir_edit2.setFixedHeight(35)
        dir_edit2.setObjectName('dir_edit2')
        dir_edit2.setStyleSheet(self.style)
        self.dir_edit2 = dir_edit2
        # "更改路径"子控件
        btn_change_dir2 = QPushButton()
        btn_change_dir2.setObjectName('btn_change_dir2')
        btn_change_dir2.setFixedSize(120, 35)
        btn_change_dir2.setStyleSheet(self.style)
        btn_change_dir2.setCursor(QCursor(Qt.PointingHandCursor))
        btn_change_dir2.clicked.connect(self.change_directory)  # 更换目录按钮的信号连接到更换目录的槽函数
        self.btn_change_dir2 = btn_change_dir2
        # 子控件加入子布局(水平布局)
        load_file_layout.addWidget(dir_to_decode)
        load_file_layout.addWidget(dir_edit2)
        load_file_layout.addWidget(btn_change_dir2)
        load_file_layout.setContentsMargins(0, 0, 0, 20)
        # 子布局和load_file子控件结合
        load_file = QWidget()
        load_file.setGeometry(0, 35, 540, 35)
        load_file.setFixedSize(540, 35)
        load_file.setLayout(load_file_layout)
        # 选择文件按钮布局
        choose_file = QWidget()
        choose_file.setGeometry(0, 35, 200, 35)
        choose_file.setFixedSize(200, 35)
        choose_file.setLayout(select_file_layout)
        # 子控件载入布局
        menu_layout.addWidget(choose_file)
        menu_layout.addWidget(load_file)
        # 纵向控件，包含选择文件按钮和路径输入框等
        menu = QWidget(self.tab2)
        menu.setGeometry(-10, 50, 540, 100)
        menu.setFixedSize(540, 100)
        menu.setLayout(menu_layout)
        self.gif.start()

    # 开始压缩按钮回调
    def start_coding(self):
        if not os.path.isdir(
                self.dir_edit1.toPlainText().replace(
                    self.label_filename1.text().split('.')[0] + '.huf', '')
        ):
            self.notify = Notification(message=True)  # 提示路径错误
            self.notify.show()
        else:
            self.notify = Notification()  # 加载进度提示窗口
            self.notify.show()
            print('开始压缩')
            print(self.dir_edit1.toPlainText())  # 打印路径
            print(self.label_filename1.text())  # 打印文件名，调试用
            filepath = self.dir_edit1.toPlainText().split('.')[0] + '.huf'  # 提取保存路径
            self.mythread = encode_thread(self.loadpath, filepath)  # 子线程对象实例化，默认是压缩
            self.mythread.update_progress.connect(self.update_progress)  # 子线程进度更新信号连接槽函数
            self.mythread.finished.connect(self.notice_finished)  # 子线程任务完成信号连接槽函数
            try:
                self.mythread.start()  # 开启子线程，进行压缩
            except:
                self.notify = Notification(message=True)
                self.notify.show()

    def update_progress(self, value):
        print("this is amazing:", value, '%')
        self.notify.setProgress(value)  # 更新进度条值
        QApplication.processEvents()  # 立即刷新

    def notice_finished(self):
        self.notify.close()  # 任务完成把进度加载界面关闭
        self.notify = Notification(message=True, text='任务完成！！')  # 出现任务完成提示框
        self.notify.show()

    # 开启解压回调函数
    def start_decoding(self):
        if not os.path.isdir(self.dir_edit2.toPlainText()):
            self.notify = Notification(message=True)
            self.notify.show()
        else:
            self.notify = Notification()
            self.notify.show()
            print('开始解压')
            filepath = self.dir_edit2.toPlainText()
            self.mythread = encode_thread(self.loadpath, filepath, ena_decode=True)  # 使能解压功能
            self.mythread.update_progress.connect(self.update_progress)  # 连接进度更新槽函数
            self.mythread.finished.connect(self.notice_finished)  # 任务完成信号连接槽函数
            try:
                self.mythread.start()  # 开始解压
            except:
                self.notify = Notification(message=True)
                self.notify.show()

    # 更换目录函数回调
    def change_directory(self):
        sender = self.sender()  # h获取信号发总者对象
        savepath = QFileDialog.getExistingDirectory(self, '选择路径', '.')  # 获取路径选择框输入的路径
        if savepath == '':
            print('\n取消选择')  # 选择为空直接退出
            return
        if sender.objectName() == 'btn_change_dir1':  # 如果来自按钮1，也就是压缩页面的信号
            filename1 = self.label_filename1.text().split('.')[0]  # 获取相应文件名
            if filename1 == '':
                filename1 = 'filename'
            self.dir_edit1.setPlainText(savepath + '/' + filename1 + '.huf')  # 生成压缩后的保存路径，并显示
        elif sender.objectName() == 'btn_change_dir2':  # 如果是来自解压页面，直接生成保存路径，并显示
            self.dir_edit2.setPlainText(savepath)
        print(savepath)
        print('更改目录')

    # 选择文件函数回调
    def select_file(self):
        sender = self.sender()  # 获取发送者对象
        filepath, _ = QFileDialog.getOpenFileName(self, '选择文件', '.', 'All Files(*);;Text Files(*.txt)')
        self.loadpath = filepath  # 保存文件读取路径
        if filepath == '':
            print('\n取消选择')
            return
        [filename, filetype] = (filepath.split('/')[-1]).split('.')  # 获得文件名和文件类型
        savepath = (filepath.split('.')[0]).replace(filename, '')  # 默认原文件所在文件路径作为保存路径
        if sender.objectName() == 'btn_select_file1':  # 来自页面1，也就是压缩
            self.label_filename1.setText(filename + '.' + filetype)  # 显示文件名
            self.dir_edit1.setPlainText(filepath.split('.')[0] + '.huf')  # 设置保存路径，并显示
        elif sender.objectName() == 'btn_select_file2':
            self.label_filename2.setText(filename + '.' + filetype)
            self.dir_edit2.setPlainText(savepath)
        # print(filepath)
        # print(filename)
        # print(filetype)
        # print('选择文件')


# 提示框对象
class Notification(QDialog):
    def __init__(self, message=False, text='文件或路径错误！'):
        super(Notification, self).__init__()  # message参数控制是进度加载提示框还是单纯的文本提示框，text用于设置文本提示框的内容
        self.message = message
        if message:  # 如果是文本提示框
            self.setupUi_message(text)  # 进行文本提示框的初始化
        else:
            self.setupUi()  # 进行进度加载提示框的初始化

    def setupUi(self):  # 进度加载提示框初始化
        self.setWindowTitle('正在努力工作...')
        self.setWindowIcon(QIcon(':/image/flower.ico'))
        self.setGeometry(0, 0, 300, 300)
        self.setFixedSize(300, 300)
        self.gif = QMovie(':/image/bgi2.gif')
        label = QLabel(self)  # 动图进度加载
        label.setMovie(self.gif)
        label.setFixedSize(300, 300)
        label.setScaledContents(True)
        label.setContentsMargins(0, 0, 0, 0)
        self.label = label
        proBar = QProgressBar(self)  # 进度条
        proBar.setObjectName('proBar')
        proBar.setGeometry(20, 260, 260, 15)
        proBar.setFixedSize(260, 15)
        proBar.setStyleSheet("text-align: center;")  # 进度条文本居中
        proBar.setValue(0)
        self.proBar = proBar
        self.gif.start()
        self.center()

    def setupUi_message(self, text):  # 文本消息提示框
        self.setWindowTitle('提示框')
        self.setWindowIcon(QIcon(':/image/flower.ico'))
        self.setGeometry(0, 0, 200, 75)
        self.setFixedSize(200, 75)
        label = QLabel(self)
        label.setFixedSize(200, 75)
        label.setText(text)
        label.setAlignment(Qt.AlignCenter)
        font = QtGui.QFont()
        font.setPointSize(14)  # 设置字体大小
        label.setFont(font)
        label.setScaledContents(True)
        label.setContentsMargins(0, 0, 0, 0)
        self.label = label
        self.center()

    def setMessage(self, str):  # 设置文本函数
        if self.message:  # 文本消息框被使能才能修改文本
            self.label.setText(str)
        else:
            return

    def setProgress(self, value):  # 设置进度条进度
        if self.message:
            return
        else:
            self.proBar.setValue(value)

    def center(self):  # 窗口居中函数
        screen = QDesktopWidget().screenGeometry()
        size = self.geometry()
        self.move((screen.width() - size.width()) / 2,
                  (screen.height() - size.height()) / 2)


class encode_thread(QThread):  # 子线程类
    update_progress = pyqtSignal(int)  # 向主线程发送当前进度信号
    finished = pyqtSignal()  # 向主线程发送任务完成的信号

    # 初始化参数，inputfile:件载入路径;outputfile:文件压缩完后的保存路径;ena_decode:是否使能编码功能
    def __init__(self, inputfile='.', outputfile='.', ena_decode=False):
        super(QThread, self).__init__()
        self.inputfile = inputfile
        self.outputfile = outputfile
        self.ena_decode = ena_decode

    def run(self):  # 子线程函数，子线程类只有该函数运行在子线程
        self.encoder = HuffmanEncoder()  # 实例化HuffmanEncoder()对象
        self.encoder.progress.connect(lambda x: self.update_progress.emit(x))  # 给HuffmanEncoder()对象的进度信号发送设置对应的槽函数
        if self.ena_decode:
            self.encoder.decodefile(inputfile=self.inputfile, outputfile=self.outputfile)
        else:
            self.encoder.encodefile(inputfile=self.inputfile, outputfile=self.outputfile)
        self.finished.emit()


if __name__ == "__main__":
    app = QApplication(sys.argv)
    main = Ui_MainWindow()
    main.show()
    sys.exit(app.exec_())
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

全部代码：

[GitHub - Kakaluoto/Huffman_code_decode](https://github.com/Kakaluoto/Huffman_code_decode)
