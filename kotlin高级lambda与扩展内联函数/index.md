# Kotlin高级：lambda与扩展内联函数

内联扩展函数是啥玩意儿？答：Kotlin为你打包好的工具包。
如果说Java是一条龙服务，那Koltin就是在一条龙之外还提供了“内联函数”这种免费附赠服务。就好比你买膜，Java只给了你膜，而Kotlin还给了你贴膜工具包，让贴膜的过程无比丝滑。丝滑如下：

# lambda简化
写过Android的小伙伴肯定认识这样一位软妹```OnClickListener```，
用Java写是这样的
```
button.setOnClickListener(new OnClickListener {
    @Override
    public void OnClick(View v) {
        // to do
    }
})
```
用Kotlin写是这样的
```
button.setOnClickListener(object:OnClickListener{
    override fun OnClick(v: View?) {
        // to do
    }
})
```
然后呈上我们的“Kotlin+lambda”魔幻大法，它就变成了这样
```
button.setOnClickListener { v-> 
    // to do
}
```
如果你连OnClick的参数View在to do时都用不到，你甚至可以不写
```
button.setOnClickListener {
    // to do
}
```
不仅如此，想用的时候，可以用it来代替参数View（因为只有一个参数，所以可以用it替代，因为it是单参数的隐式名称）。这就像曾经有一份爱情View在你面前，你没有去表达，当你回头想起，你还可以用it来替代View所留下的空白（所以View只是过客，it才是真爱）。
```
button.setOnClickListener {
    println(it.id)
}
```
相信很多小伙伴已经懵了，这是什么操作？这叫“一夫一妻制”（我乱起的，别瞎记），如果**一个接口只有一个抽象方法**，你就可以享受这种服务，多个抽象方法不行。毕竟一夫一妻的时候，你喊一下老婆，谁都知道你在喊谁，但是在一夫多妻制下，呵，还真不知道你喊的是哪一位老婆...
如果这个抽象方法有多个参数怎么办？来，让我们继续魔幻，拿```OnEditorActionListener```举例
```
textView.setOnEditorActionListener( object: TextView.OnEditorActionListener {
            override fun onEditorAction(v: TextView?, actionId: Int, event: KeyEvent?): Boolean {
                // to do something
  }
})
```
他有三个参数v: TextView?, actionId: Int, event: KeyEvent?，取参数名拿逗号隔开即可
```
textView.setOnEditorActionListener { v, actionId, event ->
            // to do something
}
```

# let
写过ES6标准JS的童鞋是不是很熟悉。
作者：你懂我意思吗？
童鞋：我懂你意思。
（眉目传情ing）
作者：不，你不懂。
let有Kotlin里没用来声明块级作用域内的变量的作用。var在JS里是花花公子，毫无规矩到处沾花惹草，但在Kotlin里已经变成了专注于自身块级作用域的好男人，好男人一个就够了，所以let就去干其他事儿了。什么事儿？简化单对象连续操作代码。
我们经常要初始化一些对象的属性，就会出现下面这种连写的情况
```
student.setId(id)
student.setName(name)
student.setAge(age)
grade = student.grade
```
每次初始化设置属性，我们都要写个studnet.然后后接方法，太累了。于是let就闪亮登场了，有了let，你可以用它画个圈圈，然后用it指代你要操作的对象，在这个圈圈里完成对**公有属性**和**公有方法**对操作。
```
student.let {
    it.setId(id)
    it.setName(name)
    it.setAge(age)
    grade = it.grade
}
```
这是什么魔力？没错，这就是爱的魔力，我们可以称之为“转圈圈”（这也是我乱起的，别瞎记）。
你还可以加个问号，用于**判空**，当student为空的时候，就不会执行let中的代码
```
student?.let {
    it.setId(id)
    it.setName(name)
    it.setAge(age)
    grade = it.grade
}
```
至于返回值，let会以闭包的形式返回返回值。你可以中途用**return表达式**，中途没执行return的话会返回**最后一行**。
```
var result = let {
    it.setId(id)
    it.setName(name)
    it.setAge(age)
    grade = it.grade
    
    1 // 前面没执行return，则最后一行为作为返回值，这里返回值为1
})
```

# with
你以为let的操作已经最骚了？不，没有最骚，只有更骚。
with不仅可以连操，还可以省略it。
```
var result = with(student, {
    setId(id)
})
```
不过不能像let那样判空。

# run
你以为with的操作已经最骚了？Again，没有最骚，只有更骚。
在let和with面前，run就是个儿子。我不是说他菜，恰恰相反，他遗传了父母的优良基因，融合了let的语法和with的神仙操作，可承受男女混合双打，可称之为“塞尔维亚妖王”（我真的是乱起的，你们别瞎记啊）。
run的语法和let一样，可以判空
```
student.run {} // 不判空
student?.run {} // 判空
```
圈圈里的写法跟with一样：不需要it。
```
var result = student?.run {
    // to do something
    1 // 最后一行为返回值，返回值为1
}
```
综上所述，run比let和with更好用，如果你们的技术总监同意的话，你甚至可以肆意地使用run来替代let和with。相对于let他不需要it，相对于with他增加了判空，实乃集两者之精华。

# apply
如果说run是let和with的爸爸的话，那么apply就是run的兄弟。这俩语法一样，唯一不同的是返回值。run比较花心，各种返回值都能返回；apply比较专情，只返回自身对象。没错，无论你怎么搞他，他都只返回自身对象。
所以run几乎啥场景都能用，而apply比较适合对象修改后又赋值给自身的场景。

# also
also的语法和let一样，但是also返回的是当前对象，而let返回的是闭包。是不是跟apply和run的关系很像。

# 总结
let，返回闭包，可判空，要用it
with，返回闭包，不可判空，省略it
run，返回闭包，集let与with之大成，可判空，省略it
apply与run一样，但是apply返回当前对象；also与let一样，但是also返回当前对象。
本章结束，完结撒花花～



