title: "Editor.commit()与Editor.apply()区别"
date: 2015-11-18 23:23:06
categories: android
tags: [android, Editor]
toc: true
description: Editor.commit()与Editor.apply()区别

---

1. apply没有返回值而commit返回boolean表明修改是否提交成功

2. apply是将修改数据原子提交到内存, 而后异步真正提交到硬件磁盘, 而commit是同步的提交到硬件磁盘，因此，在多个并发的提交commit的时候，他们会等待正在处理的commit保存到磁盘后在操作，从而降低了效率。而apply只是原子的提交到内容，后面有调用apply的函数的将会直接覆盖前面的内存数据，这样从一定程度上提高了很多效率。

3. apply方法不会提示任何失败的提示。由于在一个进程中，sharedPreference是单实例，一般不会出现并发冲突，如果对提交的结果不关心的话，建议使用apply，当然需要确保提交成功且有后续操作的话，还是需要用commit的。