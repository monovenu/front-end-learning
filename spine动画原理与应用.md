>本文首先介绍如何在前端程序中应用Spine动画以及它的原理，然后介绍如何拓展Spine动画的功能和实现内存优化。最后提出了在业务中的实现方法。
现在，让我们的页面动起来！

作者：刘洋鑫

# 1、简介
Spine 是一种针对游戏的 2D 骨骼动画。设计师制作出Spine动画后，开发人员可以在程序中使用，丰富页面的视觉效果。
Spine 通过将图片绑定到骨骼上，然后再控制骨骼实现动画。  

作为2D 骨骼动画，Spine相对于传统的逐帧动画有以下优势：
更小的体积： 帧动画需要提供每一帧图片。而 Spine 动画只需要少量的图片资源，并把骨骼的动画数据保存在一个 json 文件里面，它所占用的空间非常小。  
流畅性： Spine 动画使用差值算法计算中间帧，让动画总是保持流畅的效果。  
装备附件： 图片绑定在骨骼上来实现动画。可以方便的更换图片满足不同的需求。  
Spine官方提供了众多运行库。在项目中，我们在3d 场景中播放Spine动画，使用spine-threejs 运行库。  
![](https://github.com/monovenu/front-end-learning/blob/master/images/spine.png)

# 2、Spine源文件
设计师提供给我们的spine动画资源包括三个文件: .atlas, .json, .png。  
atlas文件包含对png图片的引用以及使用的骨骼节点的元数据。  
json文件包含骨骼信息，动作信息，节点使用的贴图数据等。  
动作数组animation中给出了骨骼各节点的具体动作：rotate, scale, shear,和translate。  
一个Spine动画可以包含多个动作，此时动作数组有多个成员。  
png文件是动画使用的雪碧图。  
# 3、应用Spine动画
应用一个Spine动画的步骤如下：  
      1.  加载从json文件中导出骨架数据  
      2.  由骨架数据生成SkeletonJson  
      3. 由SkeletonJson生成SkeletonData  
      4. SkeletonData生成SkeletonMesh  
      5. 对SkeletonMesh的动画state成员设置动画参数  
      6. 播放动画, 对SkeletonMesh进行更新  
## 3.1、生成SkeletonData
由骨架数据可以生成SkeletonData
骨架结构为Skeleton(骨架)-->Bone(骨头）-->Slot(节点）-->Attachment(附件)
图片是附件的一种，节点可以有多个附件，但同一时间只能显示一个。

骨架涉及的数据包括两个  
一是骨架的拓扑结构（连接、父子关系）  
二是骨架的各种pose，也就是每个动作对应的整个骨架的位置信息。  
蒙皮则表达的是依附在骨骼上的顶点的信息.  

骨骼绑定的过程就是确定每个顶点受哪几根骨骼的影响，每根骨骼影响的权重有多大，譬如肘部的皮肤可能同时受大臂和小臂两根骨头的影响，而远离手肘的部分可能就只受小臂骨头影响。
一般在3D骨骼动画里，每个顶点最多支持4-8根骨骼同时作用，这就已经可以很精确地表达整个蒙皮的效果了。

## 3.2、生成SkeletonMesh
在spine-three的实现中，存在数据对象和实例对象。数据对象被用于创建骨架等实例对象。  
数据对象是无状态的，可在任意数量的骨架实例间共用。
数据对象类名称以“Data”结尾，如SkeletonData。  
各实例对象有与其数据对象相同的类名称，但没有“Data”后缀。例如，SkeletonData是数据对象，而Skeleton是实例对象。
通过SkeletonData生成Skeleton，最后形成SkeletonMesh。  
Spine在3d场景中的以SkeletonMesh(一种3d 物体)呈现。  它包括
| 成员        | 功能           |
| ------------- |:-------------:|
|  batches | 脊柱 | 
|  skeleton | 骨骼，包含bone(骨头)和slot(节点)数据 | 
|  clipper | 动作数据 | 
|  state | 即AnimationState，用于调整骨架姿势实现动画 | 
|  vertices | 顶点数据 | 
|  quaternion | 父骨骼的四元数旋转值 | 
|  parent | 父节点，加入场景中后为scene | 
## 3.3、设置动画
AnimationState.setAnimation方法设置播放哪一个骨骼动画。  
setAnimation只能播放一种动画，当要连续播放不同的动画时，使用addAnimation方法，它继续播放不同的动画。 
骨骼以层级方式排列，每个骨骼受父骨骼影响，一直到根骨骼。例如，当一个骨骼旋转时，所有子骨骼及其子骨骼也会旋转。  
每个骨骼都有自己的变形方式  
    x 和 y：位置坐标  
    rotation：旋转  
    scaleX 和 scaleY：比例  
Spine动画就是通过改变这些数据来实现各种动作。
## 3.4、播放动画
实际是对skeletonMesh进行更新， 包括  
    AnimationState.update(); 逐个播放动画序列  
    skeleton.setTimelineData  旋转、缩放、变形  
    updateGeometry 更新uv贴图, 光 。  
# 4、功能拓展
项目中实现了三种任务：LoadSpineTask(加载文件)、SpineTask(播放)、Spine3dTask(移动mesh)。  
LoadSpineTask用于加载Spine资源文件。  
SpineTask用于播放动画。  
为实现更加细粒度的控制，SpineTask实现了以下功能  
|  参数 | 功能 | 
| ------------- |:-------------:|
|  callback | spineTask结束后调用的函数 | 
|  outerCallback | 用于返回mesh和spineTask | 
|  save | 是否保存创建过的SkeletonMesh | 
|  reuse | 是否保存SkeletonData | 
|  pace | 播放速率 | 
|  start | 动画起始播放时间 | 
|  keep | 播放完毕后是否从场景中移除SkeletonMesh | 
|  position | SkeletonMesh在3d场景中的位置 | 
|  scale | 大小 | 
|  inter | 是否在播放中插入其它任务 | 
|  interTime | 插入其它任务的时间点 | 
|  loopTime | 动画循环播放时间 | 

在项目中，3d场景的camera存在缩放，所以需要outerCallback返回mesh，实时的改变mesh的scale。  
有的动画需要循环播放，此时为loopTime传入Infinity。终止时将outerCallback返回的spineTask停止。  

为实现送礼物的效果，不仅需要播放Spine动画，还需要将mesh移动到人物头像上去。此时需要在spineTask运行中间插入Spine3dTask，指定插入时间(interTime)。
Spine3dTask负责将mesh从一个位置移动到另一个位置。  
播放速率和动画起始播放时间的实现是通过对setAnimation返回的TrackEntry进行自定义播放实现的。  
播放完毕后保留场景中的SkeletonMesh可以实现动画停留在最后一帧的效果。  
# 5、内存优化
为了减少内存占用，可以对多次使用的Spine动画应用两种策略。  
在同一时间点只会播放一次直接保存SkeletonMesh，下次播放时使用同一个SkeletonMesh，减少内存占用。只不过要重新设置位置、大小、播放速率、播放动作等。    
在同一时间点会同时播放多次的意味着会同时存在多个SkeletonMesh，所以保存一个SkeletonMesh不够用，需要保存生成SkeletonMesh的SkeletonData。下次播放时使用同一份SkeletonData来重新创建SkeletonMesh，无须重复加载Spine资源文件。  

当动画不再使用时，需要将它所占用的内存释放。  
首先清除Spine资源文件assets，里面占用内存最多是png图片。  
然后清除SkeletonMesh的动作资源还有mesh的重要组成部分geometry和 material，可以理解为骨骼和蒙皮。  
最后将保存过的SkeletonMesh和SkeletonData删除。即SkeletonMesh占用的内存。主要是dispose   

        var mesh = argc.mesh,
            assets = argc.assetManager.assets,
            pngSource = assets[pingResource.getSpinePng(argc.name)],
            regions = argc.atlas.regions;
        if (name) {
            assets[pingResource.getSpinePng(name)] = null;
            assets[pingResource.getSpineAtlas(name)] = null;
            assets[pingResource.getSpineJson(name)] = null;
        }

        if (pngSource && pngSource.dispose) {
            pngSource.dispose();
            pngSource.texture && pngSource.texture.dispose();
        }
        pngSource = null;
        assets = null;
        mesh.state.clearTracks();
        mesh.state = null;
        mesh.clipper.clipEnd();

        mesh.batches && mesh.batches.forEach((ele) => {
            ele.geometry.dispose();
            ele.material.dispose();
            ele.clear();
        });
        mesh.children.forEach((ele) => {
            ele.geometry.dispose();
            ele.material.dispose();
        });

        mesh.skeleton = null;
        mesh.vertices = null;
        mesh = null;

        regions.forEach((ele) => {
            ele.texture.texture.dispose();
            ele.page.texture.texture.dispose();
        });
        regions = null;
        atlas.dispose();
        if (name && this.saveList[name]) {
            this.saveList[name] = null;
        }
        if (name && this.saveSkeletonDataList[name]) {
            this.saveSkeletonDataList[name] = null;
        }

# 6、业务实现
项目中实现了两种控制器：Spine动画控制器和业务动画控制器。  
由业务动画控制器调用Spine动画控制器，外部直接调用的只有业务动画控制器。  
这两者的实例都是唯一的。播放动画时调用业务动画控制器的方法，一个特效对应一个方法。  
使用时只需传入位置等参数，无须关心Spine动画控制器的内部实现。  

Spine动画包括以下文件：  
控制器：SpineController, GameSpineController  
任务：LoadSpineTask（加载文件）、SpineTask（播放）、Spine3dTask（移动mesh)  
主要类：Spine  
资源类：SpineResource（如何加载文件）  
GameSpineController调用SpineController 外部直接调用的只有GameSpineController。  

执行流程
1、调用GameSpineController的特效方法，一个特效对应一个方法。
|  参数 | 功能 | 是否必须传入/默认值
| ------------- |:-------------:| -----:|
|  name | 三个Spine文件的名字 | 必须 |  
|  outerCallback | 用于返回mesh和spineTask | 默认null | 
|  save | 是否保存创建过的SkeletonMesh | 默认false | 
|  reuse | 是否保存SkeletonData | 默认false | 
|  pace | 播放速率 | 默认1 | 
|  start | 动画起始播放时间 | 默认0 | 
|  keep | 播放完毕后是否从场景中移除SkeletonMesh | 默认false不keep mesh | 
|  position | SkeletonMesh在3d场景中的位置 | 必须 | 
|  scale | 大小 | 必须 | 
|  inter | 是否在播放中插入其它任务 | 默认false | 
|  interTime | 插入其它任务的时间点 | inter为true时必须 | 
|  loopTime | 动画循环播放时间 | 默认0 | 

2、调用spineController的loadSpine方法
mesh save过的直接播放，否则创建spine实例并init  
3、spine init
先加载assetManager，之后运行LoadSpineTask来加载png, atlas, json文件。  
生成skeletonMesh。设置位置，加入scene  
4、SpineTask
创建SpineTask，setAnimation设置播放哪个动作，运行SpineTask，
此时spine动画真正开始播放。  

移动的特效  
运行Spine3dTask，每帧移动mesh。需要传入  
position：目标位置
pace：分几步移动过去
