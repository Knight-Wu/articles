## SSH Frequently Asked Questions

### **Sometimes my SSH connection hangs when exiting — the shell (or remote command) exits, but the connection remains open, doing nothing.**

----------

### Quick Fix

You're probably using the OpenSSH server, and started a background process on the server which you intended to continue after logging out of the SSH session. Fix: redirect the background process stdin/stdout/stderr streams (e.g. to files, or /dev/null if you don't care about them). For example, this hangs:

client% ssh _server_
server% xterm &
server% logout
_hangs..._

but this behaves as expected:

client% ssh _server_
server% xterm < /dev/null >& /dev/null &
server% logout
_SSH session terminates_
client% 

### Short Explanation

This problem is usually due to a feature of the OpenSSH server. When writing an SSH server, you have to answer the question, "When should the server close the SSH connection?" The obvious answer might seem to be: close it when the server-side user program started by client request (shell or remote command) exits. However, it's actually a bit more complicated; this simple strategy allows a race condition which can cause data loss (see the explanation below). To avoid this problem, _sshd_instead waits until it encounters end-of-file (eof) on the pipes connecting to the stdout and stderr of the user program.

This strategy, however, can have unexpected consequences. In Unix, an open file does not return eof until  _all_  references to it have been closed. When you start a background process from the shell on the server, it inherits references to the shell's standard streams. Unless you prevent this by redirecting these, or the process closes them itself (daemons will generally do this), the existence of the new process will cause  _sshd_  to wait indefinitely, since it will never see eof on the pipe connecting it to the (now defunct) shell process — because that pipe  _also_  connects it to your background process.

This design choice has changed over time. Early versions of OpenSSH behaved as described here. For some time, it was changed to exit immediately upon exit of the user program; then, it was changed back when the possibility of data loss was discovered.

### Race Condition Details

As an example, let's take the simple case of:

ssh _server_ cat foo.txt

This should result in the entire contents of the file foo.txt coming back to the client — but in fact, it may not. Consider the following sequence of events:

-   The SSH connection is set up;  _sshd_  starts the target account's shell as  _shell_  -c "cat foo.txt"  in a child process, reading the shell's stdout and sending the data over the SSH connection.  _sshd_  is waiting for the shell to exit.
-   The shell, in turn, starts  cat foo.txt  in a child process, and waits for it to exit. The file data from  foo.txt  which  _cat_  write to its stdout, however, does not pass through the shell process on its way to  _sshd_.  _cat_  inherits its stdout file descriptor (fd) from it parent process, the shell — that fd is a direct reference to the pipe connecting the shell's stdout to  _sshd_.
-   _cat_  writes the last chunk of data from  _foo.txt_, and exits; the data is passed to the kernel via the  _write_  system call, and is waiting in the pipe buffer to be read by  _sshd_. The shell, which was waiting on the  _cat_  process, exits, and then  _sshd_  in turn exits, closing the SSH connection. However, there is a race condition here: through the vagaries of process scheduling, it is possible that  _sshd_  will receive and act on the SIGCHLD notifying it of the shell's exit,  _before_  it reads the last chunk of data from the pipe. If so, then it misses that data.

This sequence of events can, for example, cause file truncation when using _scp_.


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDI2OTk5MTQ3XX0=
-->