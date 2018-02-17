---
layout: "post"
title: "Getting Started with OpenGL in Qt"
date: "2018-02-16 23:17"
author: Juntong Liu
catalog: true
tags:
  - Qt
---

## Introduction

It has been a long time since the first time I used OpenGL in this project, so you might be curious: why saying "getting started" when I have started for such a long time. Clearly, one of the reason is I don't have enough time, but more importantly, I have been changing the implementation to optimize the performance for a long time because there are really some hidden problems in the Qt-OpenGL framework. So this article is more like troubleshooting help, along with some of my personal suggestions.

## Choices

#### QOpenGLWidget

Since this article is focused on up-to-date environment, some ancient implementations like QGLWidget will not be discussed and obviously, not suggested. Instead, `QOpenGLWidget` is the way that is officially suggested. It is basically a widget that is rendered in OpenGL and it is really easy for beginners to start. However, from my experience, it will become laggy when resizing. I have not precisely located the reason, but after profiling, it might be caused by underlying buffer swapping and other caching operations. So it could be a good choice if you find no performance issue when using it.

#### QWindow

This is a very low level implementation, and believe or not, seems to be the fastest. What you get is an empty window. So you can decide how to render it. Actually there are some different methods, but here we will only discuss OpenGL. The best aspect is that you can get control of nearly everything, so there will be no unnecessary performance cost. One thing to be noticed is that `QWindow` cannot be used as a widget, because it is a window, but this function can convert it into a widget:

```cpp
QWidget::createWindowContainer(window)
```

However, the behaviour might sometimes be strange. For example, it handles mouse events in a very different way, and nothing can be drawn on top of it. Please read the [official example][1] for details. [My implementation][2] can also be used as reference. By the way, it is said that `QWindow` in OpenGL might have some problem on OSX, but since I don't have any Mac computer, so I'm unable to test it.

#### Qt Painter system

Obviously it's not an OpenGL implementation, but if your aim is do some rendering, it can also be a choice. The typical implementation of QPainter uses the raster engine. It has very few side effects and has extremely good compatibility. But the problem of raster engine is performance. Different from OpenGL implementations, in which the lag comes mainly from framework (so it could be laggy even without anything rendered), the time consumption mainly comes from the raster painting. Generally speaking, you can paint the entire screen for several times in one frame without generating lag, but if you have many overlapping shapes, it can consume your budget easily. However, for very simple rendering, I will highly suggest this way.

And possibly you don't know, `QPainter` can also use OpenGL as backend, which is said to have significant improvement in performance. Taking an OpenGL window as example, all you need to do is first, create a OpenGL-based painter:

```cpp
// Only execute once in initialization
QOpenGLContext *context = new QOpenGLContext(window);
context->create();
context->makeCurrent(this);  // Important
QOpenGLPaintDevice *device = new QOpenGLPaintDevice;
// Usually executed every frame
QPainter *painter = new QPainter(device);
```

Then you can use the OpenGL-based painter the same as normal painters, which is quite cool. But you can also use native OpenGL during the rendering:

```cpp
painter.setRenderHint(QPainter::Antialiasing, true);

painter.beginNativePainting();
// some native rendering
painter.endNativePainting();
// some painter rendering
```

The benefit of this method is you can have OpenGL-leveled performance when rendering many triangles, but still have the simplicity when rendering some tricky elements like texts. But always remember: this method could generate many strange behaviours. For example, without setting antialias at the beginning, multi-sampling of OpenGL will have no effect, and without `glDisable(GL_DEPTH_TEST)` when it is once enabled during the rendering, depth test will not work. So just get ready to different kinds of strange problems.

## Optimization

Once you have decided which way to go, don't rest too early, because there is still a long way to go. The default configuration of Qt seems not optimized in performance, or sometimes is very bad in performance. Here is some troubleshooting.

When using QOpenGLWidgets, if everything in the window moves when resizing the window, and it has a very slow response to resize operations, it might be caused by improper attributes set. The attributes control how the widget is treated. For the OpenGL widgets, these attributes might help:

```cpp
// some might be useless
setAttribute(Qt::WA_NoSystemBackground, true);
setAttribute(Qt::WA_OpaquePaintEvent, true);
setAttribute(Qt::WA_NativeWindow, true);  // most useful
setAttribute(Qt::WA_TranslucentBackground, true);
setAttribute(Qt::WA_NoSystemBackground, true);
setAttribute(Qt::WA_MSWindowsUseDirect3D, true);
```

Another thing to notice is surface format. Surface format controls the canvas attribute of the widget / window, specifically, attributes of the frame buffer, like multi-sampling and depth buffer. If it is not set correctly, some features like antialias and depth test might fail. The following code shows how to set surface format globally:

```cpp
QSurfaceFormat format;
format.setSwapInterval(0);  // for performance
format.setRenderableType(QSurfaceFormat::OpenGL);  // for performance
format.setSamples(6);  // multi-sampling
format.setDepthBufferSize(16);  // add depth buffer
QSurfaceFormat::setDefaultFormat(format);
// call before application is instantiated
```

But you can still set the format for each widget / window using very similar code. From my experience, the code given can boost the performance when resizing. But as always, please do some testing first. Another thing to notice is that `QDockWidget` will mess up the surface format, similar to the problem stated when combining QPainter and OpenGL. I would suggest using `QWindow` if it have to work with QDockWidget.

And as stated before, if you cannot solve the performance issue with `QOpenGLWidget`, just try QWindow. There should be a boost in performance.

## Other suggestions

Qt seems to treat OpenGL as an important part of the framework (although there are many problems), and there are actually many useful tools provided by Qt, for cross-platform features (you should know there are so many versions of OpenGL) and convenience. It is said native OpenGL calls (aka `gl.h`, `glu.h`) should work fine with the framework , but using functions provided by Qt might be a good choice, because there might be many minor differences on different platforms.

However, the functions are not always easy for beginners to find. For state change calls and other OpenGL core functions, you can find them in QOpenGLFunctions. To use it, you need to have your widget / window as a subclass of `QOpenGLFunctions` (which is knd of strange). But for other linear algebra related stuff, you need to find them in `QMatrix4x4`, `QVector3D` and similar classes. They will come handy when you get used to the system, but not exactly identical to native OpenGL calls.

Qt also provides a huge set of assisting classes. You can find `QOpenGLShaderProgram`, `QOpenGLContext`, `QOpenGLPaintDevice`, `QOpenGLFrameBufferObject` and so on. I will not introduce them in detail, but you should understand them easily if you are familiar with OpenGL and Qt framework.

## Conclusion

So if you read up to here, you can know how many problems I have encountered in the development of this project. Actually there are several methods to generate a window rendered by OpenGL, but it might be difficult to build a cross-platform solution, and with so many widget ready to use. Therefore, although there are still many problems, Qt might be one of few options to build a industry level application which is cross-platform and with mixed OpenGL rendering.

I hope this article can at least give an overview of the difficulty, and possibly be helpful as a reference to solve the problems. And finally, have fun programming with Qt!

[1]: http://doc.qt.io/qt-5/qtgui-openglwindow-example.html
[2]: https://github.com/PlamaDev/Plama/blob/master/gui/plot.cpp