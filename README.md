## 引子
> 这几天一直在忙一个可滑动的转盘的demo，网上也有类似的例子，但是根据老板的需求来改他们的代码，还不如重新写个完全符合需求的插件。
> 想法很美好，但是新手上路...

效果链接文末

## 需求

![image](http://or8aa6mih.bkt.clouddn.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20171115131349.png)

这个demo给的非常简单，能转动的地方有三处，内盘、外盘和指针，这三个上的集合的交集产生一个链接，通过中间的按钮跳转。

这个需求乍一看老简单老简单的，但是作为一个菜鸡第一次上道，堪比开碰碰车，头破血流。


## 分析

在做之前，也是根据自己的理解来写的旋转角度问题：
* 转盘转动的做法是：设定圆心为转动原点，动态的修改旋转角度；
* 在touchmove 计算两点与中心点的角度。

在旋转上大体上需要明白的也就这两点，但是在实际计算角度上却有很多问题。

## 弯道1之计算角度

计算角度首先要用到的一个数学方法就是反函数，在JS中表示反函数的方法有两个：

* Math.atan
* Math.atan2

说实话它们两个的区别对于本次demo真没有测出什么差异来，但是相比 atan在y特别大的时候会有误差产生的情况下，果断选择了atan2


```
(function($){
        $.fn.CompassRotate=function(options){
            var defaults={
                trigger:document,
                centerX:0,
                centerY:0,
                debug:false
            },_this=this;
            var ops=$.extend(defaults,options);
            function Init(){
                //初始化圆心点
                if(ops.centerX==0 && ops.centerY==0){
                    ops.centerX=_this.offset().left+_this.width()/2;
                    ops.centerY=_this.offset().top+_this.height()/2
                }
                $(ops.trigger).on("touchstart",function(event){
                    $(document).on("touchmove",movehandle);
                });
                $(ops.trigger).on("touchend",function(event) {
                    $(document).unbind("touchmove");
                });
            }
            //鼠标移动时处理事件
            function movehandle(event){
                var touch = event.originalEvent.targetTouches[0];
                var dis = angle(ops.centerX,ops.centerY,touch.pageX, touch.pageY);

                if(ops.debug) console.log(ops.centerX+"-"+ops.centerY+"|"+touch.pageX+"-"+touch.pageY+" "+dis);

                rotate(dis);
            }
            //计算两点的线在页面中的角度
            function angle(centerx, centery, endx, endy) {
                var diff_x = endx - centerx,
                    diff_y = endy - centery;
                var c=360 * Math.atan2(diff_y , diff_x) / (2 * Math.PI);
                c=c<=0?(360+c):c;

                return c;
            }
            //设置角度
            function rotate(angle,step){
                $(_this).css("transform", "translate3d(-50%,-50%,0) rotateZ(" + angle + "deg)");
            }
            // 指针指向角度变化和生成url
            function angleOrLink(angle) {
                Angle = angle;
            }
            Init();
        };
    })(jQuery);
    $(".box").CompassRotate({trigger:$(".box"),debug:true});
```

啰里啰嗦不如直接贴上代码，大家看得更明白些。

## 弯道2之区域集合变化

做过转盘抽奖的大佬都知道，每个奖品都对应一个角度集合，指针所转的角度[0,360]看看对应落在哪个集合上，而这个转盘也是同理，但是唯一不同的地方在于，内盘和外盘的集合是可变化的，并不是固定不变的。

```
var insideCollection = [
    {
        /* GC+S1 */
        min: 270,
        max: 360,
        reverse: false,
        mark: 's1gc'
    },
    {
        /* BC+AT */
        min: 0,
        max: 45,
        reverse: false,
        mark: 'bcat'
    },
    {
        /* BC+GT */
        min: 45,
        max: 90,
        reverse: false,
        index: 'bcgt'
    },
    {
        /* mCRC+FOLFOX */
        min: 90,
        max: 180,
        reverse: false,
        mark: 'mCRC'
    },
    {
        /* eCRC+化疗 */
        min: 225,
        max: 270,
        reverse: false,
        mark: 'eCRC1'
    },
    {
        /* eCRC+FOLFOX */
        min: 180,
        max: 225,
        reverse: false,
        mark: 'eCRC2'
    }

];
var outsideCollection = [
    {
        /* 研究 */
        min: 270,
        max: 342,
        reverse: false,
        mark: '研究'
    },
    {
        /* 指南 */
        min: 342,
        max: 54,
        reverse: true,
        mark: '指南'
    },
    {
        /* 竞品 */
        min: 54,
        max: 126,
        reverse: false,
        mark: '竞品'

    },
    {
        /* 资料 */
        min: 126,
        max: 198,
        reverse: false,
        mark: '资料'
    },
    {
        /* 机制 */
        min: 198,
        max: 270,
        reverse: false,
        mark: '机制'
    }

];
```

min，max不用说了，就是表示集合，reverse 这个属性代表的是什么呢？
在做区间划分的时候，角度的变化永远都是0-360°，“0==360”。所以，当某个集合的区间是[340,25]的时候该怎么表示呢？
当然，每次转动都有且只有一个集合会面临这样的情况，所以我用一个属性来表示这个区间跨角度了。


```
// 转盘区间分布变化
function collectionChange(angle,array) {
    array.forEach(function (ele,index) {
        ele.reverse = false;
    });
    array.forEach(function (ele,index) {
        ele.min = (Number(angle)+Number(ele.min))%360;
        ele.max = (Number(angle)+Number(ele.max))%360;
        if(ele.min > ele.max){
            ele.reverse = true;
        }
    });
    console.log(array)
}
```

*mark* 也不用多谈，选中了集合该表示表示了呀。

代码贴到这也基本完成了大体功能，最后也是在点击链接的时候根据内外盘的 *mark* 来匹配链接了：


```
$('#compass_5').on('click',function(){
    var angle = Angle;
    // 内盘标号
    var link = contrast(insideCollection) + contrast(outsideCollection);
    console.log(link);
    function contrast(array){
        var link ;
        array.forEach(function (ele,index) {
            if(angle >= ele.min%360 && angle <= (ele.max%360 ==0?360:ele.max%360)){
                link = ele.mark;
            }
            else if(ele.reverse){
                if(angle<=360 && angle >=270){
                    if(angle >= ele.min%360 && angle <= (ele.max%360 ==0?360:ele.max%360+360)){
                        link = ele.mark;
                    }
                }
                else if(angle>=0&&angle<=90){
                    if(angle+360 >= ele.min%360 && angle+360 <= (ele.max%360 ==0?360:ele.max%360+360)){
                        link = ele.mark;
                    }
                }
            }
        });
        return link;
    }
})
```

## 弯道3之坑王之王

上面说到功能大体完成了，那只是按部就班的在轮盘上只选择一个点进行转动，如果在不同位置多次转动，发现整个转盘瘫痪了——mark对应不上了。

做这个demo第一步，我是从一个简单的指针转盘开始起手的，也就是完成一个转动指针的基本操作，所以整套流程下来是可行的，因为这个指针订好了转动圆心，它的可选区域仅仅是辣么一小块，所以根本看不到为后面埋了多大坑。

### 反函数计算角度问题


```
var c=360 * Math.atan2(diff_y , diff_x) / (2 * Math.PI);
// c [-180,180];
c=c<=0?(360+c):c;
// c [0,360];
```

这样计算角度对于指针来说，没什么问题，但是对于转盘上来说可能就是个噩梦。

因为它的着落点并不确定。

导致当你点到不同区域的时候，它会给你直接将转动的角度赋值，所以会造永远是中间那条线跟着手指滑动。


![image](http://or8aa6mih.bkt.clouddn.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20171116105300.png)

[坑王之王链接(chrome调试器里查看)](https://suiyang1714.github.io/compass.github.io/)

这样的操作遇到的坑就是起始位置随着手指的变动会导致各个区域的区间也应该发生相应的变化，所以在 *touchstart* 还要进行一步操作，计算上一次结束位置与目前位置的夹角，然后再次更改区间变化。

```
$(ops.trigger).on("touchstart",function(event){
    var touch = event.originalEvent.changedTouches[0];
    var dis = angle(ops.centerX,ops.centerY,touch.pageX, touch.pageY);
    startAngle = dis;
    //再次滑动转盘后的角度与上一次结束角度不一致的情况(内盘)
    if(startAngle != ops.initAngle_in && ops.initAngle_in != 0){
        if(ops.initAngle_in>startAngle){
            insideDishAngleChangeSecondary((Number(startAngle+360)-ops.initAngle_in));
        }
        else if(ops.initAngle_in< startAngle){
            insideDishAngleChangeSecondary((startAngle-ops.initAngle_in));
        }

    }
    $(document).on("touchmove",movehandle);
});
```
## 修改后的罗盘


上一版的罗盘基本操作是将错就错，产生了一系列bug，虽然都克服了一系列bug，但还是都是在挖坑，只不过坑是平行挪动，这个坑挖不动了换了个方向继续挖而已。


### 岔路

重新审视自己的思路时，才发现自己是多么的蠢。

之前的算法是手指指在哪里，开始点为0，结束点为所指点 ，在 *touchmove* 给罗盘赋角度值时，直接将两点形成的角度赋给了罗盘 *rottate(angle)*。之后的一系列操作都是为这个地方买单，无论是重新写个函数记录变换角度在 *touchmove* 开始之前赋给罗盘分布区间、还是中心点僵硬随着手指转动。

重新思考了下罗盘的转法，有了之前的铺垫，所以思路也变得特别清晰了。
实现这个需求，记录的数据一共有三个：
* *actual_angle* ：开始点和结束点与中心点的夹角，这就是罗盘每次转动的度数，该值需要累加；
* *addAngle* ：每次转动结束后，需要给罗盘分布区间增加的值，该值等同于 *actual_angle*；
* *startAngle* ：*touchstart* 时手指着落点，即开始点。



```
$.fn.RotateH=function(options){
    var defaults={
        trigger:document,
        centerX:0,
        centerY:0,
        debug:false
    },_this=this;
    var ops=$.extend(defaults,options);
    var startAngle,addAngle,
        actual_angle = 0;
    //初始化
    function Init(){
        //初始化圆心点
        if(ops.centerX==0 && ops.centerY==0){
            ops.centerX=_this.offset().left+_this.width()/2;
            ops.centerY=_this.offset().top+_this.height()/2
        }
        $(ops.trigger).on("touchstart",function(event){
            var touch = event.originalEvent.changedTouches[0];
            var dis = angle(ops.centerX,ops.centerY,touch.pageX, touch.pageY);
            startAngle = dis;
            $(document).on("touchmove",movehandle);
        });
        $(ops.trigger).on("touchend",function(event) {

            var touch = event.originalEvent.changedTouches[0];
            var dis = angle(ops.centerX,ops.centerY,touch.pageX, touch.pageY);

            //每次转动的角度
            if(dis >=startAngle){
                //罗盘累加转动度数
                actual_angle += (dis-startAngle);
                //区间每次增加度数
                addAngle = (dis-startAngle);
            }
            else if(dis <startAngle){
                actual_angle += (dis+360-startAngle);
                addAngle = (dis+360-startAngle)
            }
            if(ops.collection) collectionChange(addAngle,ops.collection);
            else angleOrLink(dis);
            $(document).unbind("touchmove");
        });
    }
    //鼠标移动时处理事件
    function movehandle(event){

        // 获取两点之间角度
        var touch = event.originalEvent.targetTouches[0];
        var dis = angle(ops.centerX,ops.centerY,touch.pageX, touch.pageY);
        var Angle = 0;

        if(ops.debug) console.log(ops.centerX+"-"+ops.centerY+"|"+touch.pageX+"-"+touch.pageY+" "+dis);

        if(ops.pointer){
            rotate(dis);
        }
        else {
            //每次转动的角度
            if(ops.debug) {
                console.log("——————————————————————");
                console.log('上次转动的角度：'+actual_angle);
            }
            if(dis >=startAngle){
                Angle = dis-startAngle;
                if(ops.debug) {
                    console.log("转动角度："+Angle);
                    console.log("实际转动角度："+(Angle+actual_angle));
                }
                rotate((Angle+actual_angle));
            }
            else if(dis <startAngle){
                Angle = dis-startAngle+360;
                if(ops.debug){
                    console.log("转动角度：" + Angle);
                    console.log("实际转动角度："+(Angle+actual_angle));
                }
                rotate((Angle+actual_angle));
            }
        }
    }
    //计算两点的线在页面中的角度
    function angle(centerx, centery, endx, endy) {
        var diff_x = endx - centerx,
            diff_y = endy - centery;
        var c=360 * Math.atan2(diff_y , diff_x) / (2 * Math.PI);
        c=c<=0?(360+c):c;

        return c;
    }
    //设置角度
    function rotate(angle,step){
        $(_this).css("transform", "translate3d(-50%,-50%,0) rotateZ(" + angle + "deg)");
    }
    // 转盘区间分布变化
    function collectionChange(angle,array) {
        array.forEach(function (ele,index) {
            ele.reverse = false;
        });
        array.forEach(function (ele,index) {
            ele.min = (Number(angle)+Number(ele.min))%360;
            ele.max = (Number(angle)+Number(ele.max))%360;
            if(ele.min > ele.max){
                ele.reverse = true;
            }
        });
        if(ops.debug) console.log(array);
    }
    // 指针所转角度
    function angleOrLink(angle) {
        Angle = angle;
    }
    Init();
};
```

效果链接地址：[perfectCompass.github.io](https://suiyang1714.github.io/perfectCompass.github.io/.)

github 地址：[github.com/suiyang1714/perfectCompass.github.io](https://github.com/suiyang1714/perfectCompass.github.io)

