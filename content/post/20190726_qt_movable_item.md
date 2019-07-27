---
title: "Qtで移動可能なQGraphicsItemを作る"
slug: "20190726_qt_movable_item"
author: "d_yama"
date: 2019-07-26T21:37:21+09:00
draft: false
categories: ["Qt"]
tags: ["Qt", "Graphics View Framework"]
---

# はじめに
趣味で作っているアプリケーションのフレームワークとしてQtを使用しています。今回は学習の記録というかTipsというか、マウス操作で移動できるUIコンポーネントを作りたかったので、その内容をまとめます。

# マウス移動可能なQGraphicsItem
Graphics View Frameworkの全体像はQtの[オフィシャルドキュメント](https://doc.qt.io/qt-5/graphicsview.html)が参考になります。今回はGraphics SceneとGraphics Viewは組み込みのものを使用し、Graphics ItemはQGraphicsItemを継承してUIコンポーネントを作ります。

ヘッダ
```cpp
#ifndef MOVABLE_GRAPHICS_ITEM_H
#define MOVABLE_GRAPHICS_ITEM_H

#include <QObject>
#include <QGraphicsItem>
#include <QPainter>
#include <QGraphicsSceneMouseEvent>
#include <QDebug>

class MovableGraphicsItem : public QObject, public QGraphicsItem
{
    Q_OBJECT
    Q_INTERFACES(QGraphicsItem)
public:
    explicit MovableGraphicsItem(QObject *parent = nullptr);

    QRectF boundingRect() const;
    void paint(QPainter *painter, const QStyleOptionGraphicsItem *option, QWidget *widget);

protected:

signals:

public slots:

protected:
    void mousePressEvent(QGraphicsSceneMouseEvent *event);
    void mouseMoveEvent(QGraphicsSceneMouseEvent *event);
    void mouseReleaseEvent(QGraphicsSceneMouseEvent *event);
};

#endif // MOVABLE_GRAPHICS_ITEM_H
```

QGraphicsItemを継承する場合、boundingRectメソッドとpaintは純粋仮想関数なのでオーバライドが必要です。
paintは名前の通り描画を行うメソッドですが、矩形領域(QRectF)を返すboundingRectはGraphics View Frameworkにていろいろな使われ方をするようです。
ドキュメントによると、GraphicsItem同士の衝突判定などに使われるようですが、描画時にも影響があるメソッドのようです。
paintメソッドにて、boundingRectが返す矩形領域も大きな範囲を描画することができるのですが、再描画時に元のピクセルデータがクリアされるのは、このメソッドが返す矩形領域のみのようです。  

QGraphicsItemではマウス操作やキーボード操作といったイベントを受け取ることができます。
受け取り方は、各種イベントハンドラとしてメソッド(mousePressEventなど)がQGraphicsItemに定義されているので、そちらをオーバライドするだけでイベントに応じる処理を実装することができます。
今回はマウスドラッグでUIパーツを移動できるようにするため、mouseMoveEventを使用しているのですが、このイベントハンドラを有効化するにはmousePressEventとmouseReleaseEventもいっしょにオーバライドする必要があるようです。

ソース
```cpp
#include "movable_graphics_item.h"

MovableGraphicsItem::MovableGraphicsItem(QObject *parent) : QObject(parent)
{
}

QRectF MovableGraphicsItem::boundingRect() const
{
    return QRectF(-20,-20,40,40);
}

void MovableGraphicsItem::paint(QPainter *painter, const QStyleOptionGraphicsItem *option, QWidget *widget)
{
    painter->setPen(Qt::blue);
    painter->setBrush(Qt::white);
    painter->drawRect(-20, -20, 40, 40);

    Q_UNUSED(option);
    Q_UNUSED(widget);
}

void MovableGraphicsItem::mousePressEvent(QGraphicsSceneMouseEvent *event)
{
    Q_UNUSED(event);
}

void MovableGraphicsItem::mouseMoveEvent(QGraphicsSceneMouseEvent *event)
{
   this->setPos(event->scenePos());
}

void MovableGraphicsItem::mouseReleaseEvent(QGraphicsSceneMouseEvent *event)
{
    Q_UNUSED(event);
}
```

矩形を描画するようにしました。また、マウスドラッグ時に位置を更新するようにしています。
イベントハンドラに渡されるQGraphicsSceneMouseEventにはいろんな情報が入っており、ここではマウスの位置をGraphicsScene座標系で取得し、それをそのままGraphics Itemの位置としてセットしています。

 ## QGraphicsRectItemを継承して移動可能にする
 前節の実装では描画メソッドを独自に実装しましたが、描画したい形がすでに決まっているのならばQtで用意されているQGraphicsItemのサブクラスを利用するのでもよいかと思います。

 ヘッダ
```cpp
#ifndef MOVABLE_RECT_ITEM_H
#define MOVABLE_RECT_ITEM_H

#include <QGraphicsRectItem>
#include <QGraphicsSceneEvent>
#include <QPen>

class MovableRectItem: public QGraphicsRectItem
{
public:
    MovableRectItem();

    // QGraphicsItem interface
protected:
    void mousePressEvent(QGraphicsSceneMouseEvent *event);
    void mouseMoveEvent(QGraphicsSceneMouseEvent *event);
    void mouseReleaseEvent(QGraphicsSceneMouseEvent *event);
};

#endif // MOVABLE_RECT_ITEM_H

```
 ソース
```cpp
#include "movable_rect_item.h"

MovableRectItem::MovableRectItem(): QGraphicsRectItem (-10,-10,20,20)
{
    this->setPen(QPen(Qt::blue));
}

void MovableRectItem::mousePressEvent(QGraphicsSceneMouseEvent *event)
{
}

void MovableRectItem::mouseMoveEvent(QGraphicsSceneMouseEvent *event)
{
    this->setPos(event->scenePos());
}

void MovableRectItem::mouseReleaseEvent(QGraphicsSceneMouseEvent *event)
{
}
```

こっちの方がシンプルですね。

# おわりに
QtのGraphics View Frameworkを使う上で、移動可能なUIコンポーネントを作る方法をメモしました。