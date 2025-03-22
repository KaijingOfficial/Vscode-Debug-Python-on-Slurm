# Vscode-Debug-Python-on-Slurm

> This is a brief tutorial for Slurm HPC users to debug python scripts on the login node.

> Before start, make sure your vscode is attached to a slurm login node through Remote - SSH. And the `vscode-server` at login node has following vscode extensions installed: ***Python***、 ***Pylance***、 ***Python Debugger***.

> This method also works on `pytorch DDP`、 `Accelerate` and others distributed scenarios. But no more than 1 compute node.

Vscode Click on `Open Folder` -> `/path/to/your-python-repo`.
Then `Terminal` -> `New Terminal`

```bash
cd /path/to/your-python-repo
conda activate your-env # assume that you're using conda venv
pip instal debugpy
```

Add the following code to the utils of your repo, for example: `utils/debug.py`.

It will automaticly create a satisified `.vscode/launch.json`
```python
# debug.py
import debugpy
import os
import json


def setup_debug(is_main_process, port=10099):
    master_addr = os.environ['SLURM_NODELIST'].split(',')[0]
    # write to .vscode/launch.json
    launch_json_path = os.path.join('.vscode', 'launch.json')
    
    # Create .vscode directory if it doesn't exist
    os.makedirs(os.path.dirname(launch_json_path), exist_ok=True)
    
    # Default launch.json content
    default_launch_json = {
        "version": "0.2.0",
        "configurations": [
            {
                "name": "slurm_debug",
                "type": "debugpy",
                "request": "attach",
                "connect": {
                    "host": master_addr,
                    "port": port
                },
                "justMyCode": True
            }
        ]
    }
    
    # Try to read existing file, use default if fails
    try:
        with open(launch_json_path, 'r') as f:
            launch_json = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        launch_json = default_launch_json
    
    # Update the configuration
    if 'configurations' in launch_json and len(launch_json['configurations']) > 0:
        launch_json['configurations'][0]['connect']['port'] = port
        launch_json['configurations'][0]['connect']['host'] = master_addr
    
    # Write the updated json
    with open(launch_json_path, 'w') as f:
        json.dump(launch_json, f, indent=4)

    if is_main_process:  # only master process
        print("master_addr = ", master_addr, flush=True)
        debugpy.listen((master_addr, port))
        print(f"Port {port} waiting for debugger attach...", flush=True)
        debugpy.wait_for_client()
        print("Debugger attached", flush=True)
```

Call setup_debug at `your-main.py`:

```python
from utils.debug import setup_debug

is_main_process = True # This is useful in scenarios like multi-process(GPU) training
setup_debug(is_main_process)
res = 0
for a in range(0, 100): 
    res += a # -> add a break point at where you want after calling setup_debug()
```

```bash
srun bash your-launch-scripts 
# launch the python code through command line

# Command Line Output
# master_addr = XXXXXXXX
# Port 10099 waiting for debugger attach...
```

Then you can attach vscode Python Debugger through click on the `slurm_debug` button.
![](assets/button.jpg)