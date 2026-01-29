# Realtime Systems and Testing
In this document I provide some notes on how to set up realtime or lowlatency systems, in particular for robot arms. Further i describe a minimal setup for testing them.

# Realtime Systems
For many applications, realtime systems are required. These are systems that meet their timing constraints reliably under expected and worst-case conditions. 

We can differentiate between hard realtime (missing a deadline is a system failure), soft realtime (occasional deadline misses are tolerated), and firm real-time (late results are useless, but occasional misses don't crash the system) systems.

In my application, realtime is required for a robotics application controlling a robot arm. In this case, the arm is controlled through a 1000Hz control loop, so latencies below 1000μs are acceptable. 

Under Linux, a true realtime system can be achieved using a realtime kernel, that is, a Linux kernel which was built by first applying realtime patches. An excellent overview of these systems is given at https://github.com/2b-t/linux-realtime.
One very easy way to obtain such a realtime system is by using Ubuntu Pro, as described in this guide. 

Note that there are ongoing efforts to introduce more realtime and low-latency features into the Linux kernel. Whereas in prior times, one had to manually apply the realtime patches, they are already natively included in the kernel sources starting from kernel 6.12 and just need to be activated in the kernel build configuration. Building the Ubuntu kernel manually is described here: https://canonical-kteam-docs.readthedocs-hosted.com/public/how-to/build-kernel.html. When building e.g., the 6.14 kernel, in step `fakeroot debian/rules editconfigs` one has to navigate to `General setup -> Preemption Model -> Preemptible Kernel` and select the realtime or lowlatency kernel. 

# Realtime Systems and GPUs
Note that realtime systems in general don't work well with GPU drivers. In particular, the NVIDIA GPU drivers are not supported on realtime systems. This is a hard constraint in principle, since GPUs are generally optimized for maximum throughput. For this, some considerations have to be made, opting for non-deterministic calculations that have no hard time guarantees, which maximize throughput. For this reason, current GPUs don't work well with realtime systems on a fundamental level. 

So, it might not be possible to install a realtime system with a GPU. What are the alternatives?
1. One can install the realtime system together with the unsupported GPU drivers. There are often configurations that still work; however, in principle one loses the realtime guarantees in this setup. This is in most cases done by passing the `IGNORE_PREEMPT_RT_PRESENCE=1` environment variable to the NVIDIA driver installation (see https://github.com/2b-t/linux-realtime)
2. One can split up the system into two computers that communicate with each other: a simple one that is running a realtime kernel and has no GPU, and another computer that runs the GPU without a realtime kernel. Such setups are used e.g., in [deoxys](https://github.com/UT-Austin-RPL/deoxys_control) and [Droid](https://droid-dataset.github.io/droid/)
3. Maybe for many applications, true realtime guarantees are not completely required. In this case, using a low-latency kernel which supports GPU drivers is a very easy installation. It is my recommendation to always try this. For example with a new linux kernel, this is working very well for me to control a Franka Emika Panda robot arm. 

# Lowlatency kernel installation for soft realtime
The commands to install a lowlatency system for control of a Panda robot arm are shown in [lowlatency-install.md](lowlatency-install.md)


# Testing realtime capabilities / performance
There are many ways to test realtime capabilities in systems. As a baseline, a quick and dirty test of the latencies under stress should be a good start in many cases. In general, we can use cyclictest to test the latencies. This can be combined with running stress tests (e.g., stress-ng / gpu-burn), to introduce some artificial loads into the system. In good low-latency / realtime systems, these artificial loads should have no negative influence on the latencies of the test processes running with higher priorities. This is inspired by https://github.com/2b-t/linux-realtime/blob/main/doc/RealTimeOptimizations.md

The code for plotting a cyclictest graph is taken from: https://gist.github.com/HowardWhile/b1da972d77fe71b2fa210bbb7542ac09

To run the testing, run:

    ./cyclictest-hist-plot.sh

You can specify the time in the line:

    sudo cyclictest --latency=750000 -D1h -m -Sp90 -i200 -h400 -q  > output

for example by supplying `-D30m` for 30 minutes instead of `-D1h` for one hour. After running for the specified time, the script should produce a latency plot in plot.png.

For stress testing, we use `stress-ng` and `gpu_burn`, which can be obtained from https://github.com/wilicc/gpu-burn. Just follow the instructions there to build.

To test the setup under stress, run the additional CPU and GPU stress tests in three separate terminals:

    stress-ng -c $(nproc)
    ./gpu_burn -tc 3600 # Select a time that is at least as long as the cyclictest
    # Open Firefox and start some YouTube video
    ./cyclictest-hist-plot.sh

In the folder `cyclictest_results`, some example plots for different systems with different kernels are shown. 
In general, in such a plot, all the latencies should be clustered at the low end for a good realtime / lowlatency system. There should not be many spikes in the higher areas, and in particular, the maximum latency should be lower than the one required for the particular application. 

We tested on two PCs:
- panda1gpu: Intel(R) Core(TM) i7-9700 CPU @ 3.00GHz and NVIDIA RTX 2080 Ti
- panda1gpu5090: AMD Ryzen 7 9800X3D and NVIDIA RTX 5090
And tried different kernel versions. 

You can observe that the kernel without RT/lowlatency has many more high-latency spikes. In particular, under stress it has higher latencies overall. 
However, even the generic kernel is quite good with max-latencies ~400μs, which speaks to the very good performance of current hardware and kernels.
There are not many differences between the different kernels, low-latency vs. realtime and different PCs, so for our application of controlling a robot arm they should perform roughly equally.

# Testing the communication with the Panda robot:
- Connect the PC to it
- Run the RT communication test in multipanda_ros2 Docker container:
```bash
docker run -it --rm -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix --net=host --privileged multipanda_ros2
~/Libraries/libfranka/bin/communication_test 172.16.0.2
```
- Do a total of 10 runs on each system and record the number of control loops not executed
- On the new system with not RT but only low-latency kernel, we have to modify libfranka to ignore that full RT is not available:
- In robot.h change:

```cpp
explicit Robot(const std::string& franka_address,
               RealtimeConfig realtime_config = RealtimeConfig::kIgnore,
               size_t log_size = 50);
```

**RESULTS** (number of control loops not executed):
- On old system (ubuntu20-5.15-rt): 44, 57, 43, 60, 36, 41, 40, 45, 44, 50 → mean: 46  
- On new system (ubuntu24-6.11-lowlatency): 51, 54, 55, 48, 54, 48, 31, 41, 46, 30 → mean: 45.8  

→ No notable difference  
→ Low-latency kernel is good to go



