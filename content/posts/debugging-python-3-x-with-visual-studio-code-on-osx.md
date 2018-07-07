+++
author = "Jeppe Lund Andersen"
categories = ["Visual Studio", "Visual Studio Code", "Python", "Debugging"]
date = 2016-05-07T10:02:16Z
description = ""
draft = false
slug = "debugging-python-3-x-with-visual-studio-code-on-osx"
tags = ["Visual Studio", "Visual Studio Code", "Python", "Debugging"]
title = "Debugging Python 3.x with Visual Studio Code on OSX"

+++

OSX comes with Python 2.x by default, and setting the default Python version to 3.x is not without trouble. Here is how you debug Python 3.x applications with Visual Studio Code and the [Python extension](https://marketplace.visualstudio.com/items?itemName=donjayamanne.python).

1. Open the application folder in VS Code
2. Create a new launch configuration based on the Python template
3. Add `"pythonPath": "/Library/Frameworks/Python.framework/Versions/3.5/bin/python3"` to the configuration so that it looks similar to the following:

```json
{
    "name": "Python",
    "type": "python",
    "request": "launch",
    "pythonPath": "/Library/Frameworks/Python.framework/Versions/3.5/bin/python3",
    "stopOnEntry": true,
    "program": "${file}",
    "debugOptions": [
        "WaitOnAbnormalExit",
        "WaitOnNormalExit",
        "RedirectOutput"
    ]
}
```

And start debugging! 
