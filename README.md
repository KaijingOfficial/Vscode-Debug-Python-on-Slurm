# 🐛 Debugging Python on Slurm with VSCode

> **Prerequisites Checklist**  
> ✅ VSCode connected via **Remote-SSH** to Slurm login node  
> ✅ Required VSCode extensions installed:  
> &nbsp;&nbsp;&nbsp;• Python  
> &nbsp;&nbsp;&nbsp;• Pylance  
> &nbsp;&nbsp;•  Python Debugger  
> ✅ Works for distributed scenarios (PyTorch DDP/Accelerate) - *Single node only*

---

## 🚀 Getting Started

1. **Project Setup**  
   ```bash
   # In VSCode Terminal
   cd /path/to/your-python-repo
   conda activate your-env  # Activate your virtual environment
   pip install debugpy      # Install debugger dependency

2. **Debug Helper Setup**
Create utils/debug.py with this smart configuration wizard:

    ```python
    # utils/debug.py
    import debugpy
    import os
    import json

    def setup_debug(is_main_process, port=10099):
        master_addr = os.environ['SLURM_NODELIST'].split(',')[0]
        
        # Auto-configure .vscode/launch.json
        launch_json_path = os.path.join('.vscode', 'launch.json')
        os.makedirs(os.path.dirname(launch_json_path), exist_ok=True)
        
        default_config = {
            "version": "0.2.0",
            "configurations": [{
                "name": "slurm_debug",
                "type": "debugpy",
                "request": "attach",
                "connect": {
                    "host": master_addr,
                    "port": port
                },
                "justMyCode": True
            }]
        }
        
        # config updater
        try:
            with open(launch_json_path, 'r') as f:
                existing_config = json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            existing_config = default_config
        
        if existing_config.get('configurations'):
            existing_config['configurations'][0]['connect'].update({
                "port": port,
                "host": master_addr
            })
        
        with open(launch_json_path, 'w') as f:
            json.dump(existing_config, f, indent=4)

        if is_main_process:  # 🎯 Master process handler
            print(f"🚨 Debug portal active on {master_addr}:{port}", flush=True)
            debugpy.listen((master_addr, port))
            debugpy.wait_for_client()
            print("🔗 Debugger linked!", flush=True)
    ```

## 🧩 Integration in your code

1. **In your main script (your-main.py):**


    ```python
    from utils.debug import setup_debug

    # Initialize debug hook
    setup_debug(is_main_process=True)  # Listen on master process

    # Your code here
    res = 0
    for a in range(0, 100):
        res += a  # Set breakpoints at where you want, after setup_debug() is called    
    ```

2. **Launch via Slurm:**

    ```bash
    srun bash your-launch-script.sh
    # Expected output:
    # 🚨 Debug portal active on XXXXXXXX:10099
    # 🔌 Port 10099 waiting for connection...
    ```


3. **Click the debug icon in VSCode sidebar (⇧⌘D)**
4. **Select `slurm_debug` configuration**

    ![Select slurm_debug configuration](assets/button.jpg)

5. **Hit F5 to attach debugger**
