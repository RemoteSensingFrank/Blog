---
title: CAD二次开发问题集中解决
date: 2017-09-07 20:47:52
tags: 开发
categories: 学习
---
使用了一段时间的AutoCAD后遇到了很多问题，感谢一些网上的大神给我提出了很多宝贵的意见和指导，虽然有时候会很麻烦，有时候会让我绞尽脑汁，有时候会很抓狂但是问题总是在一步一步的解决，当然啦对于新的帮助最大的还是万能的百度，但是有时候网上的东西写的比较简略或者由于版本过于陈旧而缺少了参考的价值，而自己获得了很多帮助所以也想着写一些东西出来为后来学习的人做贡献。  
好了废话不多说，直接上干活，说说自己在开发的过程中所面临的问题以及自己的解决方式，当然我提出的解决方式可能不是最好的额，不过也可以让大家作为参考：  
* 问题1：在选取对象的过程中如果在一个Block中则无法获取对象的所有点，只有边界点的问题：

对于问题1也一度让我无比纠结，很多时候在CAD中为了方便我们往往会将实体建立为一个Block，然后直接操作这个Block，这样可以极大的简化操作，可是在我们使用代码获取这个实体就变得有些困难了，我们对实体的操作往往要操作到底层的点和线上，所以我们希望能够获取Block中每一个点，最简单的想法就是把Block炸开，可是也有可能出现Block嵌套的情况，所以这个炸开的过程需要递归的进行，整个炸开过程的代码为：  
```c++
public void GetPointsInEntity(Database db,Transaction trans,BlockTable bt,Entity ent,List<Point3dCollection> pointCollections)
{
    if (ent is BlockReference)
    {
        //递归获取所有块的实体
        BlockReference brf = ent as BlockReference;
        DBObjectCollection ents = (DBObjectCollection)GetEntitiesInBlock(db, trans, brf); //这句是得到块中的实体
        foreach (var obj in ents)
        {
            Entity ent1 = obj as Entity;
            GetPointsInEntity(db, trans, bt, ent1, pointCollections);
        }
    }
    else
    {
        Point3dCollection pnt3ds = new Point3dCollection();
        IntegerCollection intcollect = new IntegerCollection();
        IntegerCollection geoIndex = new IntegerCollection();
        ent.GetGripPoints(pnt3ds, intcollect, geoIndex);
        pointCollections.Add(pnt3ds);
    }
}
```
上面的代码就是将所有的块炸开然后将线的信息保存到一个点列中，我们可以看到整个递归的过程，通过不停的判断实体是否是Block，如果是则炸开，如果不是则直接获取信息，这样我们能够获取实体所有信息。ok知道了这个相信大家都能够很好的获取Block，下面我们要面临这样一个问题，如果实际上我只是想获取点信息并不是真的想将Block炸开，这可如何是好，实际上也很简单，我们在完成以上工作后不需要commit提交，则炸开操作不会执行，而我们又获取了所有块的信息。
* 问题2：实现AutoCAD中的Panel拖动，靠边效果：

实际上在处理过程中我并没有想过要实现这样的效果，但是领导看完之后觉得我做的界面太简陋因此要求提供一个这样的效果，为此我在网上找了不少资料，最后解决了这个问题，解决方案为，首先建立一个自定义的界面，界面没有边框可以嵌入Panel中，然后AutoCAD提供了一个Panel，将自定义的界面嵌入其中就可以了，具体代码如下：
```c++
Autodesk.AutoCAD.Windows.PaletteSet ps = new Autodesk.AutoCAD.Windows.PaletteSet("导线绘制");
 AutoWire constructControl = new AutoWire();

 ps.Visible = true;

 ps.Style = Autodesk.AutoCAD.Windows.PaletteSetStyles.ShowAutoHideButton | Autodesk.AutoCAD.Windows.PaletteSetStyles.ShowCloseButton;
 ps.Dock = Autodesk.AutoCAD.Windows.DockSides.None;
 ps.MinimumSize = new System.Drawing.Size(250, 450);
 ps.Size = new System.Drawing.Size(250, 450);
 ps.Add("导线绘制", constructControl);

 ps.Visible = true;
```
下面分析上述代码，第一行为新建一个PaletteSet，这个空间是AutoCAD提供的控件，通过此控件能够构造一个停靠窗口，第二行是新建一个自定义控件，然后设置停靠窗的属性，最后我们讲自定义的控件添加到这个PaletteSet中，完成我们的构造过程，构造的结果为：
![效果图](https://lh3.googleusercontent.com/-omMUHiqpZR8/WbFBAPoHjcI/AAAAAAAACVc/1TnnUh0WOgAEt0AH65yJfMcNHf31CdoyACLcBGAs/s0/QQ%25E6%2588%25AA%25E5%259B%25BE20170907152331.png "QQ截图20170907152331.png")
从上图我们可以看到，实际通过以上操作我们已经构造了一个可以停靠额窗口了，可是实际上窗口依然不好看，请原谅我无法言说的审美观。
* 问题3：如何在CAD中点击一个点获取这个点的坐标？

这个问题也是随着需求的变化而产生的，不过这也是一个十分有意义的问题，在进行CAD操作中往往存在着我自己点几个点，然后需要按照一定的算法自动构造图形的过程，而这个过程首先要面临的一个问题就是如何在CAD中点击一个点并获取这个点的坐标，实际上这也不是一个多么困难的功能但是网上的资源同样存在着不全面或者有些的博客给出的代码根本就没有经过测试的问题，下面是我的解决方案：
```c++
public Point3d MainOptFunc_Point()
{
    Point3d pnt = new Point3d(-1,-1,-1);

    Document doc = Autodesk.AutoCAD.ApplicationServices.Application.DocumentManager.MdiActiveDocument;
    Database db = doc.Database;

    Editor ed = doc.Editor;
    PromptPointResult acPointPromot = ed.GetPoint("获取点的坐标");
    if (acPointPromot.Status == PromptStatus.OK)
    {
        Transaction tran = doc.TransactionManager.StartTransaction();
        using (tran)
        {
            pnt = acPointPromot.Value;
            double x = pnt.X;
            double y = pnt.Y;
            double z = pnt.Z;
        }
    }
    return pnt;
 }
```
实际上很简单，返回值就是我们鼠标点击获取的点，比较选取实体和选点的不同我们发现主要在于Editor中获取函数有区别，实际上将函数封装成一个接口就能够很方便的进行选点操作了。
顺便提一句，以上程序的开发环境都是VS2010 AutoCAD2012。到此为止已经把面临到的几个棘手的问题都处理了，如果大家还有什么问题可以给我发邮件提问：wuwei_cug@163.com
