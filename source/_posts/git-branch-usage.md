---
title: Git最简单的分支操作教程
date: 2018-02-15 12:13:41
tags: Git
category: 工具
---

之前那篇文章，只是简单讲了一下git最基础最简单的用法，但是git还有一个非常重要的功能就是分支，默认情况下只有一个master分支，我们可以直接在master分支开发，完全没问题，
但是当你的项目有十几个甚至几十个人同时在开发的时候，如果都使用master分支，就会非常容易出现冲突、甚至出现代码被覆盖的问题，而且上线也是个问题，你不知道哪些文件可以上，哪些不可以上，很容易把一些未经测试的代码上线，这时候就需要启用分支功能。
![](http://ww1.sinaimg.cn/large/5f6e3e27ly1fyowuwste0j20nw0bvt93.jpg)

<!--more-->

## 1. git branch
默认情况下我们都是在master分支下，我们可以使用 **git branch** 命令查看当前所在分支：
```shell
jwang@jwang:~/git_demo$ git branch
* master
```
使用 **-r** 参数可以查看远程分支的情况：
```shell
jwang@jwang:~/git_demo$ git branch -r
  origin/HEAD -> origin/master
  origin/master
```
如果需要创建分支，则只需在 git branch 加上分支的名称即可，如果你想新建一个dev分支，操作如下：
```shell
jwang@jwang:~/git_demo$ git branch dev
jwang@jwang:~/git_demo$ git branch
  dev
* master
```
这里可以看到我们已经创建了一个dev分支，但是这时候我们还在master分支，并没有切换到dev分支。

## 2. git checkout
这个命令之前说过，但是在分支里面它还有另一个功能，那就是切换分支，比如如果你想切换到dev分支，用法如下：
```
jwang@jwang:~/git_demo$ git checkout dev
Switched to branch 'dev'
jwang@jwang:~/git_demo$ git branch
* dev
  master
```
这时候我们所有的pull，commit，push操作都是在当前dev分支，并不影响master分支，可见分支一大好处就是隔离代码，开一个分支写啥都行，不会影响其它人。

但是有一点需要注意，当你在dev分支使用 **git push** 推代码的时候你可能会遇到下面这个问题：
```shell
fatal: The current branch dev has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream origin dev
```
这个报错的意思是当前分支没有上游分支，什么意思呢？之前说过这个，在默认情况下，git push使用simple模式，只会把代码推送到你 **git pull** 拉取代码的分支上，但是这是远程服务器并没有dev分支，我们只是在本地创建了这个dev分支。

但是这个很容易解决，我们只需要这么做 ```git push origin dev``` 就可以：
```shell
jwang@jwang:~/git_demo$ git push origin dev
Username for 'https://github.com': wangbenjun@gmail.com
Password for 'https://wangbenjun@gmail.com@github.com': 
Counting objects: 3, done.
Delta compression using up to 12 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 282 bytes | 0 bytes/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
remote: 
remote: Create a pull request for 'dev' on GitHub by visiting:
remote:      https://github.com/wangbjun/git_demo/pull/new/dev
remote: 
To https://github.com/wangbjun/git_demo
 * [new branch]      dev -> dev
```
不过为了方便以后提交代码，我们可以使用 ```git push --set-upstream origin dev``` 命令设置上游分支，这样我们在使用 **git pull**、**git push** 命令的时候就不会报错了，它会默认跟踪dev分支。

>第一次使用git的人会很好奇这个**origin**到底是啥意思？按我的理解，这个origin其实就是指远程分支，```git pull origin dev```命令就是从远程的dev分支上拉代码。当然你可以在从master或者其它分支拉取代码，不过一般不建议从其它远程分支拉代码。

## 3. git merge
当你在这个dev分支完成开发，测试也没问题了，你就需要把这个dev分支合并到master分支，这时候就需要使用merge命令，这个命令需要明白是把谁合并到谁。假如你在dev分支执行 ```git merge master```，这就表示是把master分支合并到dev，最终代码在dev上。有些新手会理解错为把dev分支合并到master，这点需要注意。

在哪个分支上面合并都一样，你也可以在master分支上合并dev，反正最终都是一份代码，但是从项目管理的角度来说，应该先在dev分支合并master，然后再测试，因为master分支可能已经有别人提交的新的修改，你需要把这些修改合并过来。

>说到分支就不得不说到冲突，这是很多新人最害怕的事情，所谓冲突就是2个人在不同分支改动了同一行代码，这时候git就懵逼了，我到底保留哪一份呢？按提交时间先后顺序？最靠谱的方式当然是把冲突留给合并代码的人解决。

有很多新人不知道怎么解决冲突就直接把别人写的代码覆盖掉了。。。这样的事情很常见，虽然git有历史记录，代码丢是丢不了，但是解决冲突确实是个非常棘手的事情。

为了解决冲突，你必须对你所写的代码了解，同时需要和另外一个修改代码的人沟通，2个人协商一下最后保留哪些代码，千万不能一意孤行。讲道理，如果一个项目结构分层合理，同时你又是经常pull代码的话，冲突是很少见的。

```shell
jwang@jwang:~/git_demo$ git merge master
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.
```
解决冲突的方式其实不不难，使用图形化界面工具最方便，如果你不使用，你只需要找到发生冲突的文件，一般内容会如下：
```shell
# git_demo
git demo

This is a Test!

function add($a, $b)
{
    return $a+$b;
}

<<<<<<< HEAD
echo "This is a dev!";

$a = add(1,2);

var_dump($a);
=======
nothing to say
>>>>>>> master
```

请注意 <<<<<<< HEAD ...code... =======  ...code... >>>>>>> master 这3个标识中间的代码，其中上半段代码表示的是目前分支的代码，下半段表示的是master分支的代码。

你只需根据自己的需求删掉不需要的代码，保留需要的就行，比如说在这个例子里面，我只想删掉var_dump，我就可以这样改：
```shell
# git_demo
git demo

This is a Test!

function add($a, $b)
{
    return $a+$b;
}

echo "This is a dev!";

$a = add(1,2);

nothing to say
```
然后重新commit就行，最后如果没问题的话push就行。
>一般情况下，如果你不解决冲突的话是不允许你push代码的，但是你可以强制push，这样就会把冲突的代码（其实就是上面带着<<<<<符号的代码）push到远程分支，这样当然是不好滴，千万不要干这种坑事。

## 总结：
在实际开发中，我们一般遵循大概这样的流程，比如小A和小B现在要开始做一个项目的大功能，这个功能开发周期比较长，这时候由小A创建开发分支，小A的操作如下：
```shell
1.小A首先切换到master分支： git checkout master
2.然后更新代码： git pull
3.创建功能分支： git branch -b new_feature
4.提交分支到远程服务器供小B拉取： git push origin new_feature
5.小B拉取功能分支： git checkout new_feature && git pull
6.期间小A和小B共同开发，不停的pull和push
7.功能开发完成，测试完成后合并到master分支，解决可能出现的冲突
8.切换到master分支，合并dev，最后提交代码到远程仓库，如果没问题的话就可以上线了
```
这是最简单的一个分支用法，可以保证一组人在同一个分支开发，同时不会影响线上的代码。对于复杂的项目我建议可以参考 **git flow** 模型的用法，更加专业合理。

