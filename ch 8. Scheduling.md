## (Below chapter is currently under work)

# Scheduling

We have already gone over how we can get an interrupt to be triggered in our kernel at a scheduled time duration. We are now going to utilize this new feature to implement a process scheduling system. As we know, your computer CPU can usually only run a single process at a time for *each CPU core*. It only has one program counter register and is able to track only a single "thread" of instructions in memory at once. Then how is it possible that processes in memory seem to be able to run over a hundred processes simultaneously? 

The answer like we briefly discussed during chapter 6 is context scheduling. The kernel, before letting a program run, schedules the Timer interrupt to go off. 

(to be continued...)
