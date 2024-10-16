---
date: 2024-09-16
categories:
  - Python
  - Qt
  - Testing 
  - Ci
tags:
    - Python
    - Testing
    - Article
---

# Preventing segfaults in test suite that has Qt Tests

[Updated 2029.10.09 with information about "Detect leaked widgets"]

## Motivation

When providing a GUI application one needs to select a GUI backend. 
If it is a Python application that needs to work on all popular OSes[^1],
Qt is a good choice. It is a cross-platform GUI toolkit that
has good python bindings[^2].

However, Qt objects require special care during testing. 
In this post I will describe my experience of writing such tests based on 
my work on [PartSeg](https://partseg.github.io/) 
and [napari](https://napari.org/).

<!-- more -->

## The problem

As Qt is a C++ library it does not know about Python memory management.
This means that if Qt keeps a reference to some widget it does not increase the reference
count of python objects. This can lead to situations when the Python object is deleted,
but there are still events pending in the Qt event loop that reference the object.

When this happens Qt will try to access the deleted object, leading to access of unallocated memory (segfault).
This is very hard to debug because segfault can occur in any subsequent
test, making it unclear what the cause is.

The error messages vary across different operating systems:

1. Windows `Windows fatal exception: access violation` 
2. Linux `Segmentation fault (core dumped)` or `Fatal Python error: Aborted` 
3. macOS `Fatal Python error: Segmentation fault`

Moreover, this behavior is non-deterministic and may not be reproducible locally. 
One of the observed sources of difference is that the CI runs on a server version of the OS. 
I have encountered cases where I cannot reproduce the error on my development machine, 
but I could on my server. 

### What is segfault

A segfault occurs when a program tries to access memory that is not allocated to it.
In such situations, the OS will kill the program to prevent corruption. 
For security reasons, the OS does not allow handling this error, as it may be caused by malicious code.

An even worse scenario is when the addressed memory is allocated for a different object than the original pointer[^3] was pointing to.
This can lead to unpredictably modifying a different object, causing the test or program to fail unexpectedly.

## How to prevent it

This section is based on my experience and may not be complete.

### Ensure that all qt widgets are scheduled for deletion

All Qt objects have a [`deleteLater`](https://doc.qt.io/qt-6/qobject.html#deleteLater) method that schedules the object for deletion. 
This allows for the safe deletion of the object, ensuring that all pending events are processed.

If you use some widget in your test that is not a child of any other widget, 
you should call `deleteLater` on it before the test ends.
It is also good practice to ensure that the widget is hidden before deletion. 
So if your test requires showing the widget (e.g. screenshot test) you should hide it before deletion.

When using `pytest` for testing I suggest using the `pytest-qt` plugin.
This plugin provides a `qtbot` fixture that can be used to interact with Qt objects.
It also provides a [`qtbot.add_widget`](https://pytest-qt.readthedocs.io/en/latest/reference.html#pytestqt.qtbot.QtBot.addWidget) method that ensures `deleteLater` is called on the widget when the test ends.

If your widget requires special teardown you can use the `before_close_func` argument of the `add_widget` method.

### Ensure all timers, animations and/or threads are stopped

I have observed that not stopping [`QTimer`](https://doc.qt.io/qt-6/qtimer.html), 
[`QPropertyAnimation`](https://doc.qt.io/qt-6/qpropertyanimation.html), [`QThread`](https://doc.qt.io/qt-6/qthread.html) or [`QThreadPool`](https://doc.qt.io/qt-6/qthreadpool.html) can lead to a segfault. It may also lead to some other problems with your test. 

So if you use any of these objects in your test you should ensure that they are stopped before the test ends.


### Use the smallest possible widgets for tests

The process of setup and teardown of complex widgets is complex, time-consuming, and may contain bugs that are hard to detect.
So if the test purpose is to check the behavior of some widget it is better to only create this widget, not the whole window that contains it.


## How to debug and prevent

In this section, I will describe my tricks used to debug and prevent segfaults. 
However, it may not fit all projects.

### Run test under `gdb` or `lldb`

If you could reproduce the segfault locally you can run the test under `gdb` or `lldb`.
Then you could go through the stack trace and see what is the cause of a segfault.

There is also an option to increase interpolation between `gdb` and python [https://docs.python.org/3/howto/gdb_helpers.html](https://docs.python.org/3/howto/gdb_helpers.html).

You may also build Qt in debug mode and compile your Python wrapper against it. It will provide more information in the stack trace, but is complex and time-consuming.


### Prevent `QThread` and `QTimer` from running

Commonly, tests do not need to use threads. However, an integration test may trigger some threads. 
It may be a good idea to fail the test if there is a call of the `QThread.start` method. I use the following pytest fixture to do this:

```python
@pytest.fixture(autouse=True)
def _block_threads(monkeypatch, request):
    if "enablethread" in request.keywords:
        return

    from pytestqt.qt_compat import qt_api
    from qtpy.QtCore import QThread, QTimer

    old_start = QTimer.start

    class OldTimer(QTimer):
        def start(self, time=None):
            if time is not None:
                old_start(self, time)
            else:
                old_start(self)

    def not_start(self):
        raise RuntimeError("Thread should not be used in test")

    monkeypatch.setattr(QTimer, "start", not_start)
    monkeypatch.setattr(QThread, "start", not_start)
    monkeypatch.setattr(qt_api.QtCore, "QTimer", OldTimer)
```

As you may see, there is an option to allow thread usage by using the custom `enablethread` marker. 
The documentation for declaring custom markers is available in [pytest documentation](https://docs.pytest.org/en/stable/example/markers.html#registering-markers).

As the documentation does not provide examples for `pyproject.toml` I will provide examples how to do this:
```toml
[tool.pytest.ini_options]
markers = [
    "enablethread: Allow to use thread in test",
    ...
]
```

You may also spot the `monkeypatch.setattr(qt_api.QtCore, "QTimer", OldTimer)` line. It is added because `QTimer` is used internally in the `pytest-qt` plugin for `qtbot.wait*` methods.

In similar fashion, you can block usage of `QPropertyAnimation`.

This approach raises an exception when a non-allowed method is called, so it is easy to prevent unwanted usage of threads.
However, it may increase the difficulty of contributing to a project, as it is a custom behavior, which potential contributors may not expect.


### Find active timers after test end

In the napari project, we have developed a `pytest` fixture that checks if there are any active `QTimers`, `QThreads`, `QThreadPool` and `QPropertyAnimation` after the test ends.

This method is not perfect as it may not be triggered at every test suite run. So problematic code may be detected after a long time.

```python
@pytest.fixture(auto_use=True)
def dangling_qthreads(monkeypatch, qtbot, request):
    from qtpy.QtCore import QThread

    base_start = QThread.start
    thread_dict = WeakKeyDictionary()

    def start_with_save_reference(self, priority=QThread.InheritPriority):
        """Thread start function with logs to detect hanging threads.

        Saves a weak reference to the thread and detects hanging threads,
        as well as where the threads were started.
        """
        thread_dict[self] = _get_calling_place()
        base_start(self, priority)

    monkeypatch.setattr(QThread, 'start', start_with_save_reference)

    yield

    dangling_threads_li = []

    for thread, calling in thread_dict.items():
        try:
            if thread.isRunning():
                dangling_threads_li.append((thread, calling))
        except RuntimeError as e:
            if (
                'wrapped C/C++ object of type' not in e.args[0]
                and 'Internal C++ object' not in e.args[0]
            ):
                # object was deleted
                raise

    for thread, _ in dangling_threads_li:
        with suppress(RuntimeError):
            thread.quit()
            qtbot.waitUntil(thread.isFinished, timeout=2000)

    long_desc = (
        'If you see this error, it means that a QThread was started in a test '
        'but not terminated. This can cause segfaults in the test suite. '
        'Please use the `qtbot` fixture to wait for the thread to finish. '
    )

    if len(dangling_threads_li) > 1:
        long_desc += ' The QThreads were started in:\n'
    else:
        long_desc += ' The QThread was started in:\n'

    assert not dangling_threads_li, long_desc + '\n'.join(
        x[1] for x in dangling_threads_li
    )
```

It is a simplified version of the napari fixture. 
You can see the full version in [napari contest](https://github.com/napari/napari/blob/15c2d7d5ae7c607e3436800328527bd62c421896/napari/conftest.py#L444)

For other problematic objects, you can use a similar approach. There are proper fixtures in the same [`conftest.py`](https://github.com/napari/napari/blob/15c2d7d5ae7c607e3436800328527bd62c421896/napari/conftest.py) file.


### Detect leaked widgets

!!! note
    If your test suite is small it may be much simpler to review all tests and check if all top-level widgets are scheduled for deletion.

With big test datasets, it may be hard to detect if some widget is not scheduled for deletion. 

This whole section describes a set of heuristics that may help to detect such widgets, but may also lead to false positives.
If you use some custom, complex procedure for widget deletion you may need to adjust these heuristics or meet strange errors.
This heuristic may report some widgets after many test suite runs. It means that in the previous test suite runs, this widget was deleted by the garbage collector, but in this run it was not.

!!! note 
    If you are not an expert in Qt and Python I strongly suggest not to write custom teardown procedures for widgets 
    and just use `qtbot.add_widget` method everywhere.

#### `QApplication.topLevelWidgets`

Qt provides the method [`QApplication.topLevelWidgets`](https://doc.qt.io/qt-6/qapplication.html#topLevelWidgets) that returns a list of all top level widgets.
It is a nice place to start searching for leaked widgets. However, it has some limitations:

1. It may create new python wrappers for widgets, so all methods that are monkeypatched or properties defined outside `__init__` method may not be available.
2. Not all top level widgets are top level widgets that require teardown setup. For example, it returns `QMenu` objects that represent the main window menu bar.
3. It returns all top level widgets, not only those that are created in the test.


Based on the above info we cannot use custom attributes to mark widgets as handled without defining them in aconstructor. 
However, all Qt Objects have the `objectName` property that is stored as a C++ object and is not recreated in the python wrapper, though
it also could be used by custom code or stylingand therefore it is not perfect.

For code simplicity we will use the `objectName` property to mark handled widgets.
We will do this by subleasing the `QtBot` class from the `pytest-qt` plugin and overriding the `addWidget` method.

We will use the fact that `qtbot.addWidget` allows for adding a custom teardown function that will be called before widget is deleted. 
It is done by providing the `before_close_func` argument to the `addWidget` method. So if the object added to the `qtbot` 
has the `objectName` set to some value it could be changed in the `before_close_func` function.

We also need to define our own `qtbot` fixture that will use our custom `QtBot` class.

```python
from pytestqt.qtbot import QtBot

class QtBotWithOnCloseRenaming(QtBot):
    """Modified QtBot that renames widgets when closing them in tests.

    After a test ends that uses QtBot, all instantiated widgets added to
    the bot have their name changed to 'handled_widget'. This allows us to
    detect leaking widgets at the end of a test run, and avoid the
    segmentation faults that often result from such leaks. [1]_

    See Also
    --------
    `_find_dangling_widgets`: fixture that finds all widgets that have not
    been renamed to 'handled_widget'.

    """

    def addWidget(self, widget, *, before_close_func=None):
        # in QtBot implementation, the `add_widget` method is just calling `addWidget`
        if widget.objectName() == '':
            # object does not have a name, so we can set it
            widget.setObjectName('handled_widget')
            before_close_func_ = before_close_func
        elif before_close_func is None:
            # there is no custom teardown function,
            # so we provide one that will set object name

            def before_close_func_(w):
                w.setObjectName('handled_widget')
        else:
            # user provided custom teardown function,
            # so we need to wrap it to set object name

            def before_close_func_(w):
                before_close_func(w)
                w.setObjectName('handled_widget')

        super().addWidget(widget, before_close_func=before_close_func_)


@pytest.fixture
def qtbot(qapp, request):  # pragma: no cover
    """Fixture to create a QtBotWithOnCloseRenaming instance for testing.

    Make sure to call addWidget for each top-level widget you create to
    ensure that they are properly closed after the test ends.

    The `qapp` fixture is used to ensure that the QApplication is created
    before, so we need it, even without using it directly in this fixture.
    """
    return QtBotWithOnCloseRenaming(request)
```

!!! note
    As I expect that many readers of this blog post may be maintainers of napari plugins, 
    the code below contains parts specific to the napari project. These are marked with a comment.
    If you are not a napari plugin maintainer, you can remove these parts.

The fixture below is implementing our heuristic to detect leaked widgets.
It looks for all top level widgets that are not children of any other widget and have not been renamed to `handled_widget`.
It then raises an exception with a list of such widgets.


```python
@pytest.fixture(autouse=True)
def _find_dangling_widgets(request, qtbot):
    yield

    from qtpy.QtWidgets import QApplication

    from napari._qt.qt_main_window import _QtMainWindow

    top_level_widgets = QApplication.topLevelWidgets()

    viewer_weak_set = getattr(request.node, '_viewer_weak_set', set())
    # viewer_weak_set is used to store weak references to napari viewers
    # it is required if you use `make_napari_viewer` fixture in your tests

    problematic_widgets = []

    for widget in top_level_widgets:
        if widget.parent() is not None:
            # if it has a parent, then it is enough to schedule the parent for deletion
            continue
        if (
            isinstance(widget, _QtMainWindow)
            and widget._qt_viewer.viewer in viewer_weak_set
        ):
            # this if is for napari viewer created using 
            # make_napari_viewer fixture
            continue

        if widget.__class__.__module__.startswith('qtconsole'):
            # this is for jupyter qtconsole
            # we do not found yet how to properly handle some of widgets in it
            continue

        if widget.objectName() == 'handled_widget':
            continue

        problematic_widgets.append(widget)

    if problematic_widgets:
        text = '\n'.join(
            f'Widget: {widget} of type {type(widget)} with name {widget.objectName()}'
            for widget in problematic_widgets
        )
    
        for widget in problematic_widgets:
            # we set here object name to not raise exception in next test
            widget.setObjectName('handled_widget')

        raise RuntimeError(f'Found dangling widgets:\n{text}')

```
'
<!-- TBA https://github.com/napari/napari/pull/7251 -->


## Bonus tip

### Tests hanging due to nested event loop

Your tests are hanging, but the above solutions did not help. What can you still do?

One of the possible reasons is that your code created some nested event loop by opening [`QDialog`](https://doc.qt.io/qt-6/qdialog.html) 
or [`QMessageBox`](https://doc.qt.io/qt-6/qmessagebox.html) using the `exec` method. 
To get an error message instead of hanging test I use the following pytest fixture:

```python
import pytest

@pytest.fixture(autouse=True)
def _block_message_box(monkeypatch, request):
    def raise_on_call(*_, **__):
        raise RuntimeError("exec_ call")  # pragma: no cover

    monkeypatch.setattr(QMessageBox, "exec_", raise_on_call)
    monkeypatch.setattr(QMessageBox, "exec", raise_on_call)
    monkeypatch.setattr(QMessageBox, "critical", raise_on_call)
    monkeypatch.setattr(QMessageBox, "information", raise_on_call)
    monkeypatch.setattr(QMessageBox, "question", raise_on_call)
    monkeypatch.setattr(QMessageBox, "warning", raise_on_call)
    monkeypatch.setattr("PartSeg.common_gui.error_report.QMessageFromException.exec_", raise_on_call)
    monkeypatch.setattr(QInputDialog, "getText", raise_on_call)
    if "enabledialog" not in request.keywords:
        monkeypatch.setattr(QDialog, "exec_", raise_on_call)
        monkeypatch.setattr(QDialog, "exec", raise_on_call)

```

As you can see I block multiple methods that can create a nested event loop.
In some tests I need to allow calling the `exec` method of `QDialog`,
so I have defined the `enabledialog` flag that I can use to allow this call.


```python

@pytest.mark.enabledialog
def test_recent(self, tmp_path, qtbot, monkeypatch):
  ...
```






[^1]: This includes Windows, macOS, and various distributions of Linux.
[^2]: PyQt5, PySide2 for Qt5, PyQT6, PySide6 for Qt6.
[^3]: [https://en.wikipedia.org/wiki/Pointer_(computer_programming)](https://en.wikipedia.org/wiki/Pointer_(computer_programming))
