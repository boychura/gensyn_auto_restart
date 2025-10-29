🌀 **Gensyn Auto Restart Setup Guide**  
This guide walks you through editing rl-swarm to prevent data wipes, adding an automatic restart helper, and preparing your Python environment.  

⚙️ **1. Edit the Main Script**  
```cd rl-swarm && nano run_rl_swarm.sh```  
Find the following line and comment it out (add # at the start):  
```rm -r $ROOT_DIR/modal-login/temp-data/*.json 2> /dev/null  true```  
This prevents deletion of temporary data between runs.  

📦 **2. Download and Enable the Auto-Restart Script**  
Fetch the gensyn_auto.sh helper and make it executable:  
```wget -P ~/rl-swarm https://raw.githubusercontent.com/boychura/gensyn_auto_restart/refs/heads/main/gensyn_auto.sh```  
```chmod +x ~/rl-swarm/gensyn_auto.sh```  
This script monitors logs and restarts rl-swarm automatically on common runtime errors.  

🐍 **3. Create a Virtual Environment**  
Inside the rl-swarm directory, set up and activate a Python virtual environment:  
```python3 -m venv .venv```  
```source .venv/bin/activate```  
This isolates dependencies and ensures consistent runtime behavior.   

🚀 **4. Run the Auto-Restart Script**  
Once everything is set up, launch:  
```./gensyn_auto.sh```
The script will start run_rl_swarm.sh and automatically handle restarts on detected errors like:  
_BlockingIOError  
EOFError  
ConnectionResetError  
CUDA out of memory, etc._   

✅ **Done**!  
Your environment is now ready.  
You can monitor logs in /tmp/rlswarm_stdout.log or adjust settings inside gensyn_auto.sh if needed.  
