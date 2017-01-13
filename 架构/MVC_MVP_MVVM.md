# MVC
Model：模型，主要实现数据存储。<br>
View：视图，用户界面。<br>
Controller：控制器，业务逻辑。<br>
交互流程：View传递指令给Controller，Controller完成业务逻辑，更改Model状态，Model将新的数据发送到View，用户得到反馈。<br>
所有的通信都是单向的。
# MVP
将Controller改名为Presenter，同时改变通信方向。<br>
View与Model不发生联系，通过Presenter传递，View非常薄，不部署任何业务逻辑，称为"被动视图"，即没有任何主动性，而presenter非常厚，所有逻辑都部署在那里。
# MVVM
将Presenter改名为ViewModel，采用双向绑定(data-binding)，View的改动自动反应在ViewModel，反之亦然。