---
layout: post
title: 神经网络学习-使用MATLAB制作Iris数据集的分类器
author: 宋强
tags: MATLAB NN
date: 2017-01-29 11:32 +0800
---

# 数据准备
实验用到的是经典数据集中的Iris Flower Data Set，具体数据以及相关可以从维基百科中获取，参考链接：

[https://en.wikipedia.org/wiki/Iris_flower_data_set](https://en.wikipedia.org/wiki/Iris_flower_data_set)

或者直接下载已经采集好的文件：

[点击下载 Iris Flower Data Set.xlsx](../../../files/Iris&#32;Flower&#32;Data&#32;Set.xlsx)

这个数据集里包括150条花朵数据，每一条数据有四个特征参数和一个花朵种类，总共有三种花的数据，每种花50条。

这里我们采用每种花的35条数据总共105条数据作为训练样本，后面的45条数据作为测试样本，用来检验网络的输出。

# 神经网络模型设置
4个输入单元，两层隐藏单元，第一层数目不定，第二层三个隐藏单元，三个输出单元。

![](../../../images/MATLAN-Iris/NN.jpg)

MATLAB中，输入是一个四行一列矩阵，输出是一个三行一列矩阵，隐藏单元也是相应的矩阵。

# 神经网络构建
因为matlab提供了开源的神经网络库，Neural Network Toolbox，出于入门角度考虑，就用了这个开源库提供的函数提供的函数构建了神经网络。

使用这个库构建前馈神经网络非常简单，MATLAB提供了一个小示例：

```matlab
[x,t] = simplefit_dataset;
net = feedforwardnet(10);
net = train(net,x,t);
view(net)
y = net(x);
perf = perform(net,t,y)
```

这个示例演示了如何利用他的神经网络函数库构建神经网络来拟合一个数据集，其中x, t 是训练数据集的横坐标和纵坐标，feedforwardnet()函数一个参数10代表一个隐藏层，这个隐藏层有10个隐藏单元，这个函数返回了一个net类型数据，之后使用train函数，这个函数读取了x和t的维数之后自动在net中添加相应数量的输入单元和输出单元，并且进行训练，这里是按照默认参数进行训练的。

这个时候再返回的net就变成了一个函数，把我们想要预测的输入输入到这个函数里，就得到了神经网络的预测输出。

我们把105组数据集读取进matlab，同样使用这两个函数就可以得到训练好的神经网络，设置网络的相关参数，就可以改变网络的表现，尽量提升神经网络的性能。但是这里我们使用newff函数来代替feedforwardnet，可以详细设定网络参数。

![](../../../images/MATLAN-Iris/Traning&#32;samples.jpg)

三张图分别代表神经网络的三个输出， 蓝色线代表神经网络计算结果，红色线代表对这个数据进行取1或者-1的操作，绿颜色线代表实际数值。

由这张图可以知道，105组数据中有一组数据偏离了真正数据，所以

$$拟合率=\frac{104}{105}\times100\%=99.05\%$$

当这个拟合率过低的时候，就证明神经网络的构建并不好，或者训练的不好，或者数据不好，总之是有问题，需要改进。

![](../../../images/MATLAN-Iris/Testing&#32;samples.jpg)

在测试数据中我们可以看出来，45组数据中5组数据预测失败，

$$神经网络准确率=\frac{40}{45}\times100\%=88.89\%$$

当然我们可以通过增加更多的训练样本或者调整神经网络结构来优化预测结果，这一次只是第一次真正编程使用机器学习算法，优化或者结构的问题以后会更详细的讨论。

# MATLAB程序
```matlab
%bound.m
function [y] = bound(x)
    y = zeros(size(x, 1), size(x, 2));
 
    for i = 1:size(x, 1)
        for j = 1:size(x, 2)
            if x(i, j) > 0
                y(i, j) = 1;
            else
                y(i, j) = -1;
            end
        end
    end
end
 
<code>processedData = mapminmax(trainSet);
net = feedforwardnet([10 3], 'traingdx');
%net = newff( minmax(processedData) , [10 3] , { 'tansig' 'purelin' } , 'traingdx' ) ; 
net.layers{1}.transferFcn = 'tansig';
net.layers{2}.transferFcn = 'purelin';
 
net.trainParam.max_fail = 30;
net.trainparam.show = 50 ;
net.trainparam.epochs = 500 ;
net.trainparam.goal = 0.01 ;
net.trainParam.lr = 0.01 ;
 
net = train(net, processedData, trainClass);
 
output = net(processedData);
x = 1:length(processedData);
subplot(3, 2, 1);
hold on
grid on
plot(x, output(1, :), x, trainClass(1, :), x, bound(output(1, :)))
subplot(3, 2, 3);
hold on
grid on
plot(x, output(2, :), x, trainClass(2, :), x, bound(output(2, :)))
subplot(3, 2, 5);
hold on
grid on
plot(x, output(3, :), x, trainClass(3, :), x, bound(output(3, :)))
xlabel('Traning Samples')
processedTest = mapminmax(testSet);
x = 1:length(processedTest);
testOutput = net(processedTest);
subplot(3, 2, 2);
hold on
grid on
plot(x, testOutput(1, :), x, testClass(1, :), x, bound(testOutput(1, :)))
subplot(3, 2, 4);
hold on
grid on
plot(x, testOutput(2, :), x, testClass(2, :), x, bound(testOutput(2, :)))
subplot(3, 2, 6);
hold on
grid on
plot(x, testOutput(3, :), x, testClass(3, :), x, bound(testOutput(3, :)))
xlabel('Testing Samples')
```

这里要注意的是trainSet和trainClass分别是4x105和3x105的训练特征矩阵和样本分类结果矩阵，同样testSet和testClass分别是4x45和3x45的测试特征矩阵和测试分类结果矩阵，这四个矩阵都是从Excel文件中读取出来的。