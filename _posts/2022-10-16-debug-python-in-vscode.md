---
  title: Using Debug Mode in VSCode for Python Programming
  description: >-
    Debug mode configuration for python.
  # author: pointer
  date: 2022-10-16 19:01:00 +0800
  categories: [Blogging, Tutorials]
  tags: [misc]
  # media_subpath: /images/2022-08-10-ipv6-version-of-frp/
  pin: false
---


## Requirements

When programming in Python using VSCode, you may sometimes need to set breakpoints in debug mode to monitor the program's execution in real-time.

Compared to the traditional print debugging method, debug mode offers several advantages:
- **Convenient startup**: Simply click the green play button or use a shortcut to start debugging, without switching to the command line to run the script.
- **Real-time breakpoints**: You can add or remove breakpoints while the program is running.
- **Interactive programming**: Once breakpoints are set, you can use Python’s interactive mode to inspect variables or test snippets of code.
- **View variables and call stack**.
- ...


## References

> [1] [VSCode, "Debugging"](https://code.visualstudio.com/docs/editor/debugging)  
> [2] [VSCode, "VS Code Debugger not working for Python"](https://learn.microsoft.com/en-us/answers/questions/724858/vscode-debugger-not-working-for-python.html)  
> [3] [VSCode, "Python debugging in VS Code](https://code.visualstudio.com/docs/python/debugging)  


## Debug Button

By default, after installing the Python extension, a Run button appears in the top right corner of VSCode.

However, this button only runs the currently open file. When you click it, it automatically opens a terminal and executes the following command:

```sh
python ${file}
```

But in many cases, this method is not sufficient.

For example:
- You may not want to run the currently open file but a **fixed** entry file (such as `main.py`).
- You may need to pass **command-line arguments**.
- You may need to set up **multiple configurations with different arguments**.

To address these needs, VSCode provides the **Run & Debug** mode [1], which allows **custom execution settings** using the `launch.json` file.


## Prerequisite: Python Extension Version Issue

As of the time of writing (October 16, 2022), the latest version of the Python extension has **compatibility issues with VSCode's debugging mode**, causing the debug process to crash immediately without any output. To fix this, downgrade to version `v2021.12.1559732655`. 

You can follow these steps:

1. Open the Python extension page in VSCode.
2. Next to the "Uninstall" button, click the small triangle and select "**Install Another Version...**".
3. Wait for the version list to load and select `v2021.12.1559732655`.
4. **Disable automatic updates** to prevent it from updating to a newer, buggy version:
  - Open the Command Palette (Ctrl+Shift+P).
  - Run Extensions: Disable Auto Update for All Extensions.
  - Restart VSCode.

For more details, refer to [2].


## Setting Up and Using `launch.json`

### Creating a `launch.json` File

1. Click on the Run & Debug panel in the left sidebar.
2. Click "Create a launch.json file".
3. Select Python, then choose "Run Current File".

This creates a `launch.json` file inside the `.vscode` folder. Once configured, the available configurations will appear in the Run & Debug panel, allowing you to easily start debugging.

VSCode uses a JSON format to define the debug settings. For full details, check the official documentation [3].

### Common `launch.json` Fields:

- `name`: A descriptive name for the configuration, which appears in the dropdown menu.
- `program`: Specifies the script to run. Typically, you can set it to `${workspaceFolder}/main.py` to run `main.py` from the workspace root.
- `module`: Similar to program, but runs a Python module using `-m`.
- `python`: Specifies the Python interpreter path. This is useful if you need to use a specific Anaconda environment.
- `args`: A list of command-line arguments to pass to the script.
- `console`: Specifies where the program runs. The default **integrated terminal** is recommended.
- `justMyCode`: Set this to false to allow stepping into external library code.
- `env`: Environment variables, such as specifying the GPU device for training.

### An Example `launch.json` Configuration

Below is an example configuration from my setup:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "main.py",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/main.py",
      "console": "integratedTerminal",
      "justMyCode": false,
      "args": [
        "--config", "param.ini",
      ],
      "env": {
        "CUDA_VISIBLE_DEVICES": "0",
      },
    },
  ]
}
```

That's it!
