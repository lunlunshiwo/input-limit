# bug的产生和修改  
  
上周临近周末休息的时候，一个同事跑过来了，对我说：“阿伦啊，有一个页面出问题了，火狐浏览器所有的input都没法输入了。”我一听，是不是你给加了什么属性，让input输入框只读了啊。看了一下代码，很正常的一个输入框，并且CSS写的也很正常。
```
<input id="ipt-message" type="text" placeholder="请输入身高" />
```  
但是运行之后发现无法输入任何东西，包括字母、符号、数字（后来实验发现，输入了汉字之后可以输入符号和数字，这个暂时未发现原因）。那么问题就来了，肯定是js部分的问题了。当时我就猜大概是做了限制，但是当时我还是比较相信同事写的代码的。我就面对着几个js文件一个一个的看，一个几千行行代码，知道我看到了下面这段代码：
```
$("input[type='text']").keypress(function (e) {
    if (!String.fromCharCode(e.keyCode).match(/[0-9\.]/)) {
        return false;
    }
})  
```  
虽然当时没验证这段代码的情况，但是直觉告诉我找到原因了。我把这段代码复制出来还原了一下，大概意思就是所有的文本输入框在keypress事件触发这个函数，使用正则验证输入的情况，只能输入小鼠和小数点。但是当运行的时候问题出现了，bug出现了，那么问题找到了，就是这四行代码导致的，在那堆js代码中去掉这四行之后就没有了问题。  
之后我研究出错的原因时发现，e.keyCode在谷歌时正常显示，但是在火狐浏览器下就会出现问题了：

 

||谷歌浏览器|火狐浏览器|IE11浏览器|
|按键“a”|keydown：keyCode为65，charCode为0;keypress：keyCode为97，charCode为97;keyup：keyCode为65，charCode为0|keydown：keyCode为65，charCode为0;keypress：keyCode为0，charCode为97;keyup：keyCode为65，charCode为0|keydown：keyCode为65，charCode为0;keypress：keyCode为97，charCode为97;keyup：keyCode为65，charCode为0|
|按键“1”| keydown：keyCode为49，charCode为0;keypress：keyCode为49，charCode为49;keyup：keyCode为49，charCode为0|keydown：keyCode为49，charCode为0;keypress：keyCode为0，charCode为49;keyup：keyCode为49，charCode为0|keydown：keyCode为49，charCode为0;keypress：keyCode为49，charCode为49;keyup：keyCode为49，charCode为0|
|按键“Backspace”|keydown：keyCode为8，charCode为0;keypress未触发;keyup：keyCode为8，charCode为0|keydown：keyCode为8，charCode为0;keypress:keyCode为8，charCode为0;keyup：keyCode为8，charCode为0|keydown：keyCode为8，charCode为0;keypress为触发;keyup：keyCode为8，charCode为0|  
  
那么问题就找到原因了，通过String.fromCharCode(e.keyCode)是无法做到兼容火狐浏览器返回按键值，因为当输入数字和字母时，其keyCode都为0。  
因此，我修改了一下这个代码：
```
$("input[type='text']").keypress(function (e) {
    var code = e.charCode || e.originalEvent.charCode；
    if (code != 0) {
        if (!String.fromCharCode(code).match(/[0-9\.]/)) {
            return false;
        }
    }
}) 
```  
  
originalEvent是jquery对原生event属性的封装。“code != 0”这个判断是在火狐浏览器下对是否按键为“Backspace”的判断，如果没有这个判断，会导致Backspace键无法使用，无法删除这个情况的发生。

# 谈限制输入框输入类型  
  
其实无论是使用哪种方式来限制输入框的输入的类型，都离不开keyup、keypress、keyup和比较少见的textInput四个事件来触发。其中，前三个为各个浏览器共同支持的，而textInput仅有IE9+，Safari和Chrome，这也正式比较常见的浏览器（或其内核）。前三个事件为键盘事件，最后一个为文本事件。其触发的顺序为keydown(按键按下)——>keypress(按键值插入文本)——>textInput(按键值插入文本)——>keyup(按键弹起)。textInput和keypress的发生很相似，但二者还是有区别：任何可以获得焦点的元素都可以触发keypress事件，但只有可编辑区域才能触发textInput事件。textInput事件只会在用户按下能够输入实际字符的键时才会触发，而keypress事件则在按下那些能够影响文本显示的键时也会触发（比如退格键）。  
在实际的操作中，我们会发现在输入中文的时候，只用按键事件对其进行触发限制输入框的操作体验非常不好。比如上面的代码，只在keypress事件触发限制，当我们输入法切换在中文时，按下字母之后按空格键或者“shift”键，会发现我们的限制失效了。
## 原生js  

因此为了更好的体验，我们需要更多的事件触发限制，达到我们的目的。因此，我们应在触发keypress后在对即将插入文本框的值进行一边过滤。就拿上文的条件只能输入数字和小数点来说，我们需要在textIput事件发生时判断一下文本框的value值，使用正则进行一下过滤。代码如下：
```
    var EventUtil = {
        addHandler: function (element, type, handler) {
            if (element.addEventListener) { //DOM2级
                element.addEventListener(type, handler, false);
            } else if (element.attachEvent) { //DOM1级
                element.attachEvent("on" + type, handler);
            } else {
                element["on" + type] = handler; //DOM0级
            }
        },
        removeHandler: function (element, type, handler) { //类似addHandler
            if (element.removeEventListener) {
                element.removeEventListener(type, handler, false);
            } else if (element.detachEvent) {
                element.detachEvent("on" + type, handler);
            } else {
                element["on" + type] = null;
            }
        }
    }
    var textbox = document.getElementById('input');
    EventUtil.addHandler(textbox, 'textInput', function (e) {
        e.target.value = e.target.value.replace(/[^0-9\.]/g, '')
    })
```  
  
利用事件监听把绑定textIput事件，当触发时再对其value过滤一下。结果还是不错的：
但是在上文提到过，textInput属性对火狐浏览器无兼容，因此，我们就需要使用keyup对其进行代替（但效果不好）：
```
textbox.onkeyup=function (e) {
    e.target.value = e.target.value.replace(/[^0-9\.]/g, '')
}
```
## Vue的自定义指令

简单少量的input我们可以使用双向绑定后，watch到变化进行限制，如下：
```
<input id="ipt" type="text" v-model="iptVal"/>
<script>
    var qwe=new Vue({
        el:'#ipt',
        data(){
            return{
                iptVal:''
            }
        },
        watch:{
            iptVal(val){
                this.iptVal=val.replace(/[^0-9\.]/g, '')
            }
        }
    })
</script>
```  
但是，当面对大量的input输入框，我们更向往用简单的方式去解决，甚至简单到只写一个指令（比如v-limit）就可以。这样，我们就需要用到自定义指令的知识(可参考我的博客《Vue的土著指令和自定义指令》)。
这个value是随着输入不断更新的，因此，我们需要选用update这个钩子函数：
```
    Vue.directive('limit', {
        update: function (el) {
            el.onkeypress = function (e) {
                var code = e.charCode;
                if (code != 0) {
                    if (!String.fromCharCode(code).match(/[0-9\.]/)) {
                        return false;
                    }
                }
            }
            el.addEventListener('textInput', function (e) {
                e.target.value = e.target.value.replace(/[^0-9\.]/g, '')
            })
            el.onkeyup = function (e) {
                e.target.value = e.target.value.replace(/[^0-9\.]/g, '')
            }
        }
    })
```  
此时，调用的方式特别简单，只需要增加“v-limit”这个指令即可。
```
<input id="ipt" type="text" v-model="iptVal" v-limit />
```
