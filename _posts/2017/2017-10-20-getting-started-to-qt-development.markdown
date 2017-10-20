---
layout: "post"
title: "Getting Started to Qt Development"
date: "2017-10-20 09:02"
author: Juntong Liu
catalog: false
tags:
  - Qt
---

## Introduction

Well, I spent some days to figure out how to write the hello world program in Qt, specifically Qt 5. To get the source code, you can go to [Plama repository][1], clone it and checkout to commit `ce57fb08781a67e4bf9907f9b24447caff90b0df`. It's a little annoying, so I've got a copy for you, just go to [this page][2] to get it. I'm going to give a brief introduction on how to make a very simple program. Unlike the official documents, this is going to be short and simple.

## Qmake

If you are familiar with development on Linux, there are several automatic building systems, such make or cmake, for C styled languages. So apparently, Qmake is for Qt projects. Compared to other building systems, Qmake is super easy to use. You only need to specify Qt modules to use and source code position to compile (including `.cpp`, `.h`, `.ui` files) in the Qmake file. An example is given as follows:

```
TARGET = Plama  // Project name
TEMPLATE = app  // Project type

// Qt modules
QT      += core gui widgets

// Let compiler to report warnings
DEFINES += QT_DEPRECATED_WARNINGS

// Cpp source files
SOURCES += main.cpp interface.cpp

// Header files
HEADERS += interface.h

// Qt design forms
FORMS   += main.ui
```

This is a very typical project file for desktop application. By the way, one Qmake file represents a Qt project and file extension of them is `.pro`. The template value represents project type, for library projects, it is `lib`; for main project including many sub-projects, it is `subdirs'. Such information can be found at [official document][3].

But in most of the case, updating such a project could be messy for any of version control services since all the files are crowded in the top level directory. Therefore, it is better to organize them into a subdirectory, then you can easily put other documents like license or readme into the top level directory.

This can be very simple to do with Qmake. Simply sort all the project file into one folder (including the Qmake file), then create another Qmake file in the top level, and put following things:

```
// Declare the project contains sub-projects
TEMPLATE = subdirs

// Define sub-project folders
SUBDIRS += Application
```

By doing this, you can make the previous project a sub-project of this. All you need to put into the top-level directory is this Qmake file. It might be difficult to understand, but when you see the example project, it will be very clear.

# Qt Design Forms

First thing to say: It's unnecessary to read the content of these files because they are not designed for human reading. The design forms describes the interface layout, very similar to other frameworks, like xml layout for Android, or layout designer in IntelliJ for Java Swing. There is a graphical interface to help you arrange all your things. Most of the widgets should be very familiar to you if you have done GUI programming in any language.

# Source code

This should be the core part of a program, while there are not much things special in Qt programming. This project uses Qt for C++, the original implementation. So it is just C++ code, with some tweaks by Qt.

First of all, you need a `main.cpp` file, like all the C++ programs:

```cpp
#include "interface.h"
#include <QApplication>

int main(int argc, char *argv[]) {
    QApplication a(argc, argv);
    Main w;
    w.show();

    return a.exec();
}
```

The application is managed by the framework, the only thing you need to do is provide the interface, or `Main` in this the code. This is defined in `interface.h` (or whatever name you want):

```cpp
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>

namespace Ui {
class Main;
}

class Main : public QMainWindow {
    Q_OBJECT

public:
    explicit Main(QWidget *parent = 0);
    ~Main();

private:
    Ui::Main *ui;
};

#endif // MAINWINDOW_H
```

And `interface.cpp`:

```cpp
#include "interface.h"
#include "ui_main.h"

#include <Qt>
#include <QComboBox>
#include <QDebug>
#include <QLineEdit>
#include <QPushButton>
#include <QStringList>
#include <QKeySequence>

Main::Main(QWidget* parent)
    : QMainWindow(parent), ui(new Ui::Main) {
  ui->setupUi(this);

  ui->comboBox->addItems(QStringList() << "+" << "-" << "*" << "/");

  connect(ui->pushButton, &QPushButton::clicked, [this](bool) {
    auto a = ui->numA->text().toFloat();
    auto b = ui->numB->text().toFloat();
    switch (ui->comboBox->currentIndex()) {
      case 0: a += b; break;
      case 1: a -= b; break;
      case 2: a *= b; break;
      case 3: a /= b; break;
      default: break;
    }
    ui->result->setText(QString::number(a));
  });
}

Main::~Main() {
  delete ui;
}
```

Please be informed the class `Main` inside `namespace UI` is not the same as the class `Main` outside the namespace. The class inside `namespace UI` comes from the design form, which is described previously. The class `UI::Main` is implemented automatically according to the design form. Therefore, only declaration is needed here, the implementation has been done in the header files automatically.

For the `Main` class outside the namespace, most of the APIs of `QMainWindow` has been already implemented, you only need to fill the contents. Here `ui->setupUi(this)` is used to do this work. Other things you might want to do is set up the behaviors of widgets. All the variable names has been defined in the design form and can be changed according to your needs. Here I added some items to the combo box so that the user can select the operator (s)he want. Then I set up the connection to detect push button clicks, which will be described in next section.

# Signal

The signal system is a very famous system in Qt to handle all the input events, similar to event listeners in Java Swing and event bus in minecraft forge. Every object can emit signals and have sockets to receive signals. When a signal of one object is connected to a socket of another object, whenever the signal function is called, the socket function will be triggered.

This system provides help for decoupling of modules because the function calls are not invoked on specific objects. In addition, it simplifies the code because the signal emitter do not need to iterate the listeners any more. A typical usage of the signal system is:

```cpp
QObject::connect(emitter, signal, receiver, socket)
```

Although this structure is very classical in object oriented programming, it is lacking some functional support, especially when C11 is out, which provides many improvements for functional programming. Therefore, in Qt 5, another set of connection is available:

```cpp
QObject::connect(emitter, signal, function)
```

In `interface.cpp`, the second syntax is used. Whenever the button is clicked (the click signal is triggered), the lambda function will be invoked to decide further operations.

# Conclusion

Here is a screenshot of the current work. The result may vary on different platforms.

![screenshot][4]

Hopefully this article is good enough to describe my current understanding about Qt programming and help others (if any) to get started.


[1]: https://github.com/PlamaDev/Plama
[2]: https://github.com/PlamaDev/PlamaBlog/blob/master/share/2017/getting-started-to-qt-development.zip
[3]: http://doc.qt.io/qt-4.8/qmake-project-files.html
[4]: /img/posts/2017/getting-started-on-qt-development.png
