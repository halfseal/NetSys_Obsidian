---
creator: Maike
type: Hands-On
created: 2022-11-21
---
The plan: installing and setting up GPGPUSim and maybe running some tests to get a feeling for the simulator.

1. Installing GPGPU-Sim from the git repository([https://github.com/gpgpu-sim/gpgpu-sim_distribution](https://github.com/gpgpu-sim/gpgpu-sim_distribution)).
2. Installing another repository for simulations ([https://github.com/gpgpu-sim/gpgpu-sim_simulations](https://github.com/gpgpu-sim/gpgpu-sim_simulations))
3. The simulation repository requires a framework called torque, so install that, too ([http://docs.adaptivecomputing.com/torque/5-1-1/Content/topics/hpcSuiteInstall/manual/1-installing/installingTorque.htm](http://docs.adaptivecomputing.com/torque/5-1-1/Content/topics/hpcSuiteInstall/manual/1-installing/installingTorque.htm))
4. when configuring torque, add the “—disable-gcc-warnings” flag

Make for torque fails. I have been trying to fix the errors in the code

Fixes in torque:

req.cpp replace current == ‘\0’ with *current == ‘\0’