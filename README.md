# Simple-Shell-Implementation-system
Simple-Shell is a Linux based shell implementation written in C language. The shell provides a simple and efficient command line interface for users to interact with the operating system. The implementation of Simple Shell includes built-in commands and the ability to run external programs.

I. Overview
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

III. Creating a History Feature
The shell should support a history feature:

If the user types !!, the shell should repeat the last command.

For example:

If the user first types ls -l, then types !!, the shell should execute ls -l again.

If there is no command in history, the shell should display: No commands in history.

IV. Redirecting Input and Output
The shell should support input/output redirection:

Output redirection (>):

Example: ls > out.txt will save the output of ls to out.txt.

Input redirection (<):

Example: sort < in.txt will use in.txt as input for the sort command.

Use the dup2() function to redirect file descriptors:

Example: dup2(fd, STDOUT_FILENO) redirects output to a file.

Assumptions:

Commands will have either input or output redirection, but not both.

No need to handle complex cases like sort < in.txt > out.txt.

V. Communication via a Pipe
The shell should support pipes to connect commands:

Example: ls -l | less will send the output of ls -l as input to less.

Use the pipe() function to create a pipe between two processes.

Use dup2() to redirect the output of the first command to the input of the second.

Implementation:

The parent process creates a child process for the first command (ls -l).

The child process creates another child process for the second command (less).

A pipe is established between the two child processes.

Assumptions:

Commands will contain only one pipe.

Pipes will not be combined with redirection operators.

Code Structure
The provided code outlines the basic structure of the shell:

#include <stdio.h>
#include <unistd.h>
#define MAX_LINE 80 /* Maximum length of a command */

int main(void) {
  char *args[MAX_LINE / 2 + 1]; /* Command line arguments */
  int should_run = 1; /* Flag to control the shell loop */

  while (should_run) {
    printf("osh>");
    fflush(stdout);

    /* Steps:
     * 1. Read user input.
     * 2. Fork a child process.
     * 3. Execute the command in the child using execvp().
     * 4. Parent waits for the child unless '&' is used.
     */
  }
  return 0;
}
Project Tasks
Execute Commands:

Fork a child process and execute commands using execvp().

Handle the & symbol for background execution.

History Feature:

Implement !! to repeat the last command.

Display an error if no command is in history.

Input/Output Redirection:

Support > and < using dup2().

Pipes:

Implement pipes using pipe() and dup2().

Key System Calls
fork(): Creates a child process.

execvp(): Executes a command.

wait(): Makes the parent wait for the child.

dup2(): Redirects input/output.

pipe(): Creates a pipe for communication between processes.


