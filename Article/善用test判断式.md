# 善用test判断式

在前面的内容中，我们提到过 $? 这个变量所代表的意义， 此外，也透过 && 及 || 来作为前一个指令是否能够成功进行的一个参考。

那么，如果我想要知道 /test 这个目录是否存在时，难道一定要使用 ls 来执行， 然后再以 $? 来判断执行成果吗？当然不是！我们可以用 *test* 这个指令来侦测呢！

当我要*检测系统上面某些档案或者是相关的属性*时，利用 test 这个指令来工作， 真是好用得不得了，举例来说，我要检查 /test是否存在时，使用：

```
$ test -e /test
```

*执行结果并不会显示任何讯息，但最后我们可以透过 $? 或 && 及 || 来展现整个结果*。 例如我们在将上面的例子改写成这样：

```
$ test -e /test && echo "exist" || echo "Not exist"
```

最终的结果可以告知我们是 exist 还是 Not exist。

现在我们知道 -e 是测试一个东西在不在， 如果还想要测试一下其它内容，还有哪些标志可以来判断的呢？如下表所示：

![这里写图片描述](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/09/09/20210909104115.jpg)

![这里写图片描述](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/09/09/20210909104126.jpg)

![这里写图片描述](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/09/09/20210909104135.jpg)



编写shell脚本，实现这样的功能：检查当前目录是否存在名叫test的文件，如果有则输出exist，没有则创建test文件。