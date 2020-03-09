---
title: PHP平衡二叉树
date: 2016-01-05 19:00:03
category: PHP
tags: 
    - PHP
---

## 平衡二叉树

之前讲过树，二叉树，二叉排序树，现在说说这个平衡二叉树，平衡二叉树是一个平衡的二叉排序树，关键在于平衡，它的意思是其中每一个节点的左子树和右子树的高度差不多都是1。

<img src = "/images/old/5f6e3e27ly1fvg1sihqxcj20p80es0wr.jpg" />

为什么要平衡呢？还是为了提高查找速度，举个例子有一个数组 [3,2,1,4,5,6,7,10,9,8]，如果按照二叉排序树的算法生成之后应该是图1的结果，这样其实对于查找是不利的，举个例子，如果你要找节点8,
你得找7次，但是如果是图2这种结构，则只需要3次。

<!--more-->

<img src = "/images/old/5f6e3e27ly1fvg1zeac9nj20oa0a6tb6.jpg" />

下面看一下图：

<img src = "/images/old/5f6e3e27ly1fvgyvvo4xyj20lf09040f.jpg" />
<img src = "/images/old/5f6e3e27ly1fvgyw55iwlj20ji091gn6.jpg" />
<img src = "/images/old/5f6e3e27ly1fvgyw56ua0j20ob0giq78.jpg" />
<img src = "/images/old/5f6e3e27ly1fvgyw57cblj20pn0jv0x8.jpg" />
<img src = "/images/old/5f6e3e27ly1fvgyw562xgj20o00hcn1w.jpg" />

最后贴一个PHP实现的代码：
### 树节点类:
```php
class Node
{
    public $key;

    public $data;

    public $bf; //平衡因子

    public $leftNode;

    public $rightNode;

    public function __construct($key, $data)
    {
        $this->key  = $key;
        $this->data = $data;
    }

    public function __toString()
    {
        return $this->key . '--->' . $this->data;
    }
}
```
这里涉及到几个算法，比较难理解：

### 左旋和右旋

```php
/**
 * 对以p为根的二叉排序树作右旋处理
 * 处理之后p指向新的树根节点，即旋转处理之前的左子树的树节点
 * @param $p Node
 */
public function RRotate(Node &$p)
{
    $l = $p->leftNode;

    $p->leftNode = $l->rightNode;

    $l->rightNode = $p;

    $p = $l;
}

/**
 * 对以p为根的二叉排序树作左旋处理
 * 处理之后p指向新的树根节点，即旋转处理之前的右子树的树节点
 * @param $p Node
 */
public function LRotate(Node &$p)
{
    $r = $p->rightNode;

    $p->rightNode = $r->leftNode;

    $r->leftNode = $p;

    $p = $r;
}
```
### 左平衡旋转和右平衡旋转

```php
/**
 * 左平衡旋转
 * @param $root Node
 */
public function leftBalance(Node &$root)
{
    $l = $root->leftNode;

    switch ($l->bf) {
        case EH:
            $l->bf    = RH;
            $root->bf = LH;
            self::RRotate($root);
            break;
        case LH:
            $root->bf = $l->bf = EH;
            self::RRotate($root);
            break;
        case RH:
            $lr = $l->rightNode;
            switch ($lr->bf) {
                case LH:
                    $root->bf = RH;
                    $l->bf    = EH;
                    break;
                case EH:
                    $root->bf = $l->bf = EH;
                    break;
                case RH:
                    $root->bf = EH;
                    $l->bf    = LH;
                    break;
            }
            $lr->bf = EH;
            self::LRotate($root->leftNode);
            self::RRotate($root);
    }
}

/**
 * 右平衡旋转
 * @param $root Node
 */
public function rightBalance(Node &$root)
{
    $r = $root->rightNode;

    switch ($r->bf) {
        case RH:
            $root->bf = $r->bf = EH;
            self::LRotate($root);
            break;
        case EH:
            $root->bf = RH;
            $r->bf    = LH;
            self::LRotate($root);
            break;
        case LH:
            $rl = $r->leftNode;
            switch ($rl->bf) {
                case EH:
                    $root->bf = $r->bf = EH;
                    break;
                case RH:
                    $root->bf = LH;
                    $rl->bf   = EH;
                    break;
                case LH:
                    $root->bf = EH;
                    $r->bf    = RH;
                    break;
            }
            $rl->bf = EH;
            self::RRotate($root->rightNode);
            self::LRotate($root);
            break;
    }
}
```

最后是插入算法：
```php
public function insertAvl(&$root, int $key, string $data, bool &$taller = false)
{
    if (!$root) {
        $root           = new Node($key, $data);
        $root->leftNode = $root->rightNode = null;
        $root->bf       = EH;
        $taller         = true;
        return true;
    } else {
        if ($key == $root->key) {
            $taller = false;
            return false;
        }

        if ($key < $root->key) {
            //在左子树中搜索
            if (!self::insertAvl($root->leftNode, $key, $data, $taller)) {
                return false;
            }
            if ($taller) {
                switch ($root->bf) { //检查树的平衡度
                    case LH:
                        self::leftBalance($root);
                        $taller = false;
                        break;
                    case EH:
                        $root->bf = LH;
                        $taller   = true;
                        break;
                    case RH:
                        $root->bf = EH;
                        $taller   = false;
                        break;
                }
            }
        } else {
            //在右子树中搜索
            if (!self::insertAvl($root->rightNode, $key, $data, $taller)) {
                return false;
            }
            if ($taller) {
                switch ($root->bf) { //检查树的平衡度
                    case LH:
                        $root->bf = EH;
                        $taller   = false;
                        break;
                    case EH:
                        $root->bf = RH;
                        $taller   = true;
                        break;
                    case RH:
                        self::rightBalance($root);
                        $taller = false;
                        break;
                }
            }
        }
    }

    return true;
}
```
