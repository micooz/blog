---
title: 【C#算法实现】安卓QQ小游戏天天爱消除辅助。
date: 2013-08-18 00:00:00
tags:
  - tech
  - csharp
  - algorithm
---

近期腾讯在安卓手机客户端出了个小游戏——天天爱消除。初次玩这款游戏真是觉得自己脑残，玩了半天也只能靠提示进行下去。小伙伴们个个几十万让我几万的情何以堪？！之后想了想如何自动消除，于是有了这篇文。注意这里仅仅是在Windows平台实现了算法，没有应用于android环境，所以想拿现成的辅助程序请出门左拐。

算法的实现比较容易，主要是数据结构需要注意，也是一下午就搞定了，初期为了快速开发选用了C#作为实现语言。本文的示例代码将以C#给出。 

首先分析游戏，消除界面是一个7*7固定的棋盘，有有限种基本棋子(蓝，紫，红，白，咖啡，绿，橘黄)，还有一些特殊功能棋子暂不考虑，下面是一张游戏截图：

![](191415_JP2C_580940.png)

连成三个通常有几种情况可以考虑，大家在玩的时候应该能体会到，下面的6个图(注意颜色区分)是以3*2为分析单元的所有情况，对于2*3其实同理，只不过要转换一下。

![](csharp-qq-game-ttaxc-0.png)

![](csharp-qq-game-ttaxc-1.png)

下面有两个图是1*4的所有情况(4*1同理)： 

![](csharp-qq-game-ttaxc-2.png)

在消除的时候通常就是按照上述规律进行的，所以我们只要对棋盘数据进行分割，然后对应上面几种情况，判断固定位置是否是相同的棋子，那么就可以直接知道应该交换哪两个位置了。 

不同棋子可以用1234..这样简单表示，棋盘数据是一个7*7的二维数组，假设上图是这样一个二维数组：

```csharp
new int[7, 7] 
{
  {4,0,0,4,1,5,3},
  {1,2,1,0,1,0,5},
  {5,4,6,2,0,4,3},
  {2,4,6,5,1,3,5},
  {1,5,3,1,0,0,5},
  {6,5,3,2,2,1,2},
  {1,0,5,2,0,2,2}
};
```

下面对数据进行2*3、1*4划分，由于划分之后要进行判断并返回所在棋盘数组的位置，因此需要定义一个含有位置字段和数据段的一个数据结构 ，位置是通过x、y坐标来定位，也就是数组下标：

```csharp
/// <summary>
/// 定义棋子坐标(从0开始)
/// </summary>
public struct point{
    public int m_x;
    public int m_y;
    public point(int x, int y)
    {
        m_x = x;
        m_y = y;
    }
};
```

为了能够简单初始化，所以给了一个构造函数来初始化成员。

```csharp
/// <summary>
/// 每个棋子都记录一个位置
/// </summary>
private struct ele
{
    public point m_pIndex;
    public int m_data;
};
```

下面的函数将棋盘数据转化为含有位置信息的新数组,通过成员refData进行存储：

```csharp
/// <summary>
/// 将原始数据进行转化
/// </summary>
private void ConvertData() {
    refData = new ele[7, 7];

    for (int i = 0; i < 7; ++i)
    {
        for (int j = 0; j < 7; ++j)
        {
            refData[i, j].m_data = chessData[i, j];
            refData[i, j].m_pIndex = new point(i, j);
        }
    }

}
```

最重要的就是分析单元的提取，在进行提取之前，我们首先需要定义几个标志位：

```csharp
//行列计次
int s_x = 0;
int s_y = 0;
//遍历标识
bool is_2x3 = true;
bool is_3x2 = false;
bool is_1x4 = false;
bool is_4x1 = false;
bool m_bFinished = false;
```

s_x,s_y 是用于定位棋子的变量，因为取3*2和2*3的时候需要紧挨着取，保证不能漏掉。

下面4个bool变量标志当前正在取哪种类型的区域。

```csharp
/// <summary>
/// 返回下一个分析单元
/// </summary>
/// <returns></returns>
private ele[,] GetNextSection()
{
    ele[,] sec = null;

    //纵向(2*3)遍历
    if (s_x <= 5 && s_y <= 4 && is_2x3)
    {
        sec = new ele[2, 3]{
            {refData[s_x,s_y],refData[s_x,s_y+1],refData[s_x,s_y+2]},
            {refData[s_x+1,s_y],refData[s_x+1,s_y+1],refData[s_x+1,s_y+2]}
        };

        s_x++;
        //纵向到底
        if (s_x == 6)
        {
            //右移一个单位
            s_x = 0;
            s_y++;
            if (s_y == 5) {
                //遍历完毕
                s_x = 0;
                s_y = 0;
                is_2x3 = false;
                is_3x2 = true;
                is_1x4 = false;
                is_4x1 = false;
            }
        }
        return sec;
    }
    //纵向(3*2)遍历
    if (s_x <= 4 && s_y <= 5 && is_3x2)
    {
        sec = new ele[3, 2]{
            {refData[s_x,s_y]  ,refData[s_x,s_y+1] },
            {refData[s_x+1,s_y],refData[s_x+1,s_y+1]},
            {refData[s_x+2,s_y],refData[s_x+2,s_y+1]}
        };
        //为简化代码，对3*2的section进行矩阵变换
        sec = new ele[2, 3] {
            {sec[2,0],sec[1,0],sec[0,0]},
            {sec[2,1],sec[1,1],sec[0,1]}
        };

        s_x++;
        //纵向到底
        if (s_x == 5)
        {
            //右移一个单位
            s_x = 0;
            s_y++;
            if (s_y == 6)
            {
                //遍历完毕
                s_x = 0;
                s_y = 0;

                is_2x3 = false;
                is_3x2 = false;
                is_1x4 = true;
                is_4x1 = false;
            }
        }
        return sec;
    }

    //1*4遍历
    if (s_x <= 6 && s_y <= 3 && is_1x4)
    {
        sec = new ele[1, 4]{
            {refData[s_x,s_y],refData[s_x,s_y+1],refData[s_x,s_y+2],refData[s_x,s_y+3]}
        };
        s_y++;
        if (s_y == 4)
        {
            //下移一个单位
            s_y = 0;
            s_x++;
            if (s_x == 7)
            {
                //遍历完毕
                s_x = 0;
                s_y = 0;

                is_2x3 = false;
                is_3x2 = false;
                is_1x4 = false;
                is_4x1 = true;
            }
        }
        return sec;
    }
    //4*1遍历
    if (s_x <= 3 && s_y <= 6 && is_4x1)
    {
        sec = new ele[4, 1]{
            {refData[s_x,s_y]},
            {refData[s_x+1,s_y]},
            {refData[s_x+2,s_y]},
            {refData[s_x+3,s_y]}
        };
        //对4*1的section进行矩阵变换
        sec = new ele[1, 4]{
            {sec[3,0],sec[2,0],sec[1,0],sec[0,0]}
        };

        s_x++;
        if (s_x == 4)
        {
            //下移一个单位
            s_x = 0;
            s_y++;
            if (s_y == 7)
            {
                //遍历完毕
                s_x = 0;
                s_y = 0;

                is_2x3 = true;
                is_3x2 = false;
                is_1x4 = false;
                is_4x1 = false;

                m_bFinished = true;
                //return null;
            }
        }
        return sec;
    }

    return sec;
}
```

每次调用函数将返回下一个区域，这就靠自己领悟了。我觉得我写的稍微有点复杂，不过应该比较容易理解，效率上也是相当快的。 
取一个区域进行分析，这里是连续分析，直到找到一个含有可交换棋子的区域 ：

```csharp
/// <summary>
/// 返回一个可交换位置
/// </summary>
/// <returns></returns>
public point[] GetNextPoints(){
    point[] pt=null;
    ele[,] sec;
    while (pt == null && !m_bFinished)
    {
        sec = GetNextSection();
        if (sec == null) return null;

        if (sec.GetLength(0) == 1)
        {
            //有两种情况
            if (sec[0, 0].m_data == sec[0, 1].m_data && sec[0, 0].m_data == sec[0, 3].m_data)
            {
                pt = new point[2]{
                    sec[0,2].m_pIndex,
                    sec[0,3].m_pIndex
                };
            }
            else if (sec[0, 0].m_data == sec[0, 2].m_data && sec[0, 0].m_data == sec[0, 3].m_data)
            {
                pt = new point[2]{
                    sec[0,0].m_pIndex,
                    sec[0,1].m_pIndex
                };
            }
        }
        else if (sec.GetLength(0) == 2)
        {
            //只有6种可消除情况
            if (sec[0, 0].m_data == sec[0, 2].m_data && sec[0, 0].m_data == sec[1, 1].m_data
                || sec[0, 1].m_data == sec[1, 0].m_data && sec[0, 1].m_data == sec[1, 2].m_data)
            {
                pt = new point[2]{
                    sec[0,1].m_pIndex,
                    sec[1,1].m_pIndex
                };
            }
            else if (sec[0, 0].m_data == sec[1, 1].m_data && sec[0, 0].m_data == sec[1, 2].m_data
                || sec[1, 0].m_data == sec[0, 1].m_data && sec[1, 0].m_data == sec[0, 2].m_data)
            {
                pt = new point[2]{
                    sec[0,0].m_pIndex,
                    sec[1,0].m_pIndex
                };
            }
            else if (sec[0, 0].m_data == sec[0, 1].m_data && sec[0, 0].m_data == sec[1, 2].m_data
                || sec[1, 0].m_data == sec[1, 1].m_data && sec[1, 0].m_data == sec[0, 2].m_data)
            {
                pt = new point[2]{
                    sec[0,2].m_pIndex,
                    sec[1,2].m_pIndex
                };
            }
        }
    }//while

    return pt;
}
```

这个没什么技术含量，按部就班。
现在道德我们想要的两个point了，实际测试情况请看图，为了方便查看，下标进行了+1处理：

![](193505_JLNz_580940.png)

算法速度上没有严格测试，但都是瞬间完成的。
