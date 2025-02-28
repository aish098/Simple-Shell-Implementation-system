
The goal is to design a shell interface that:

Prompts the user for commands (e.g., osh>).

Executes each command in a separate child process.

Supports background execution (using &).

Implements input/output redirection and pipes for communication between commands.

The shell will work similarly to a UNIX shell. For example:

If the user types cat prog.c, the shell will display the contents of prog.c.

If the user adds an ampersand (&), like cat prog.c &, the command will run in the background, allowing the parent and child processes to run concurrently.

The project is divided into several parts:

Creating a child process to execute commands.

Adding a history feature to repeat the last command.

Supporting input/output redirection using > and <.

Implementing pipes to connect the output of one command to the input of another.

II. Executing Commands in a Child Process
The first task is to modify the main() function to:

Fork a child process using fork().

Parse the user's input into tokens (e.g., args[0] = "ls", args[1] = "-l", args[2] = NULL).

Execute the command in the child process using execvp(args[0], args).

Check for the & symbol to determine if the parent should wait for the child to finish.

For example:

If the user enters ps -ael, the shell will:

Fork a child process.

Execute ps -ael in the child.

Wait for the child to finish unless & is used.
