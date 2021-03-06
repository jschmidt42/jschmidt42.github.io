---
layout: post
title: node.js instance manager
category: project
tags: [qt, node, c++]
---

node.js instance manager is a simple application that lets you manage the execution of multiple instance of node.js applications. You can play/stop many instances and monitor them using a simple UI that reside in the system tray. You can specify the port of all instances and many other process parameters. 

Follow on [GitHub (https://github.com/jschmidt42/nim)](https://github.com/jschmidt42/nim)

## Motivation

It is useful to have a single view on all your node instances that are running and being able to quickly restart them, etc.

## Usage

![main ui](/images/main.png)

- You add a new node instance slot by pressing the **Add** button.
- Then browse for a **node.js startup script** *(same script you would start using node.exe on the command line)*.
- If your node application is a server that needs a given port, enter it in the **port field**.
- Finally press the **Play** button to start the node process. If it didn't crashed at startup, the icon should pass to a stop button state.
	- Pressing the stop button will kill the node process
- *Repeat the process for other node instances.*

### Additional options

- More node instance options...
	- ![more options](/images/more-options.png)
		- **Open Browser**: opens a browser view on http://localhost:port
		- **Open Explorer**: opens a file explorer view where the startup scripts is
		- **Edit Env. Vars**: edits environment variables that will be passed to the node process when started.
		- **Log**: opens a log window of the running node process.
		- **Debug**: adds the *debug* argument to the node process to be debugged
		- **Delete**: deletes the node instance slot 
- When you minimize the application it will be hidden from the application taskbar and a icon will be available in the system tray to re-open it.
	- ![system tray](/images/system-tray.png)

## Requirements

- Install Visual Studio 2010
- Install [Qt libraries 4.8.4 for Windows (VS 2010, 234 MB)](http://releases.qt-project.org/qt4/source/qt-win-opensource-4.8.4-vs2010.exe "QT 4.8.4")
- Install [Visual Studio Add-in 1.1.11 for Qt4](http://releases.qt-project.org/vsaddin/qt-vs-addin-1.1.11-opensource.exe)