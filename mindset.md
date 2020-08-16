# How to solve hard (technical) problems

### Mindset

- There are no hard problems. There is just lack of information about how the system works
- Remember that the bug is happening for a logical reason

- Be unreasonable confident in your ability to fix the bug
- The harder the bugs you solve the better you will be

- Every error is an opportunity to learn



### Finding the root cause of the problem

- Try to get the issue reproducible
    - Can you reproduce it from the command line?
        - It's easier for other people to reproduce the issue
        - It's easier to test the fix
- Are there any log files? What's the error message?
    - Read the error description. Every word of it. Twice.
    - Is there a typo somewhere (command line/configuration/code)?
- Isolate the problem
    - Remove some parts of the system and try to reproduce the bug
    - Vary one thing at a time while keeping all other things constant



### Issue still not fixed? Checklist

- [ ] Try to tackle hard problems in the morning with a fresh mind and without disruption (before you check mails, chat, ticket system, monitoring)
- [ ] Do you have multiple issues? Try to solve the underlying issue first (like a ssh connection that is dropped every minute)
- [ ] Is it really a problem or just a misunderstanding (works as expected?)
    - [ ] Is there a security feature/policy that blocks your work?
- [ ] Get a stable debugging environment
- [ ] Does the problem occur only on a single server? The same thing is working somewhere else?
    - [ ] What's the difference? Compare!
- [ ] When did the problem occur first? What has changed?
- [ ] Can you increase the debug log?
- [ ] Do some sanity checks 
    - [ ] Are you on the right virtual machine?
    - [ ] Can you ping the target host?
    - [ ] Is DNS still working?
    - [ ] Check network traffic with ngrep/tcpdump. Do you see what you expect?
    - [ ] Is one of the disks full?
    - [ ] Are you editing the right file?
        - [ ] Write some garbage and try to compile/a syntax check
    - [ ] Check the monitoring system
        - [ ] Do other VMs of the customer have problems?
        - [ ] Do other VMs running on the same hypervisor have problems?
        - [ ] Is the whole data center down?
    - [ ] Is the customer logged in on the system? What is he doing (check bash_history and `ps -u`)



### After some time of debugging

- Force yourself to express the problem in an easy and comprehensible way that a random teddy bear understands it
- Be patient and accept that things just take longer than expected
- Try to understand what happens. Not: endless trial and error
    - Is there documentation that can help you understanding the system
    - Talk to other people knowing the system better than you
- Take a break (go for a walk, do some exercises, take a deep breath, drink some water)
- Step back: What's  the problem? What's the reason for the problem? What's the actual goal you are trying to achieve?
- You don't have time and stuck on some unrelated details?
    - Fix the problem by not fixing the problem: use a different approach to solve your actual problem



### If you copy/paste from Stackoverflow

- Don't copy/paste from Stackoverflow without understanding the actual problem

- Don't copy/paste from Stackoverflow without understanding the proposed solution

    - If you don't have time for it right now => make a note about it (even after solving it)
    - If you don't know what the command/tool is doing read the man page [(https://explainshell.com)](https://explainshell.com/)
    - Don't copy/paste commands/code. Type it on your own

    

### After solving the issue

1) Well done! I'm glad you didn't give up!

2) What have you learned?

2) What were the wrong assumptions?

3) How can you solve a similar problem in future even faster?



### 