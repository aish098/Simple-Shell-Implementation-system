#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <time.h>
#include <errno.h>
#include <dirent.h>
#include <signal.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/msg.h>
#include <pwd.h>
#include <syslog.h>

#define MAX_LINE_LENGTH 1024
#define BUFFER_SIZE 64
#define REDIR_SIZE 2
#define PIPE_SIZE 3
#define MAX_HISTORY_SIZE 128
#define PROMPT_MAX_LENGTH 30
#define PROMPT_FORMAT "%F %T "

#define TOFILE_DIRECT ">"
#define APPEND_TOFILE_DIRECT ">>"
#define FROMFILE "<"
#define PIPE_OPT "|"


int running = 1;
char history[MAX_LINE_LENGTH] = "No commands in history";


void init_shell();
char *prompt();
void error_alert(char *msg);
void remove_end_of_line(char *line);
void read_line(char *line);
void parse_command(char *input_string, char **argv, int *wait);
int is_redirect(char **argv);
int is_pipe(char **argv);
void parse_redirect(char **argv, char **redirect_argv, int redirect_index);
void parse_pipe(char **argv, char **child01_argv, char **child02_argv, int pipe_index);
void exec_child(char **argv);
void exec_child_overwrite_from_file(char **argv, char **dir);
void exec_child_overwrite_to_file(char **argv, char **dir);
void exec_child_append_to_file(char **argv, char **dir);
void exec_child_pipe(char **argv_in, char **argv_out);
void exec_command(char **args, char **redir_argv, int wait, int res);
int simple_shell_cd(char **args);
int simple_shell_help(char **args);
int simple_shell_exit(char **args);
int simple_shell_history(char *history, char **redir_args);
int simple_shell_redirect(char **args, char **redir_argv);
int simple_shell_pipe(char **args);
void set_prev_command(char *history, char *line);
char *get_prev_command(char *history);


char *builtin_str[] = {"cd", "help", "exit"};
int (*builtin_func[])(char **) = {&simple_shell_cd, &simple_shell_help, &simple_shell_exit};

int simple_shell_num_builtins() {
    return sizeof(builtin_str) / sizeof(char *);
}


int main(void) {
    char *args[BUFFER_SIZE];
    char line[MAX_LINE_LENGTH];
    char t_line[MAX_LINE_LENGTH];
    char *redir_argv[REDIR_SIZE] = {NULL, NULL};
    int wait;
    int res = 0;

    init_shell();

    while (running) {
        printf("%s:%s> ", prompt(), getcwd(NULL, 0));
        fflush(stdout);

        read_line(line);
        strcpy(t_line, line);
        parse_command(line, args, &wait);

        if (strcmp(args[0], "!!") == 0) {
            res = simple_shell_history(history, redir_argv);
        } else {
            set_prev_command(history, t_line);
            exec_command(args, redir_argv, wait, res);
        }
        res = 0;
    }

    return 0;
}


void init_shell() {
    printf("**********************************************************************\n");
    printf("  #####                                    #####                              \n");
    printf(" #     # # #    # #####  #      ######    #     # #    # ###### #      #      \n");
    printf(" #       # ##  ## #    # #      #         #       #    # #      #      #      \n");
    printf("  #####  # # ## # #    # #      #####      #####  ###### #####  #      #      \n");
    printf("       # # #    # #####  #      #               # #    # #      #      #      \n");
    printf(" #     # # #    # #      #      #         #     # #    # #      #      #      \n");
    printf("  #####  # #    # #      ###### ######     #####  #    # ###### ###### ###### \n");
    printf("**********************************************************************\n");
    char *username = getenv("USER");
    printf("\n\n\nCurrent user: @%s\n", username);
}


char *prompt() {
    static char _prompt[PROMPT_MAX_LENGTH];
    time_t now = time(NULL);
    struct tm *tmp = localtime(&now);

    if (strftime(_prompt, PROMPT_MAX_LENGTH, PROMPT_FORMAT, tmp) == 0) {
        fprintf(stderr, "Error: Failed to format time\n");
        exit(EXIT_FAILURE);
    }

    return _prompt;
}

// Remove newline character from input
void remove_end_of_line(char *line) {
    line[strcspn(line, "\n")] = '\0';
}

void read_line(char *line) {
    if (fgets(line, MAX_LINE_LENGTH, stdin) == NULL) {
        if (feof(stdin)) {
            printf("\n");
            exit(EXIT_SUCCESS);
        } else {
            perror("Error: Failed to read input");
            exit(EXIT_FAILURE);
        }
    }
    remove_end_of_line(line);
}


void parse_command(char *input_string, char **argv, int *wait) {
    int i = 0;
    for (i = 0; i < BUFFER_SIZE; i++) argv[i] = NULL;

    *wait = (input_string[strlen(input_string) - 1] == '&') ? 0 : 1;
    input_string[strlen(input_string) - 1] = (*wait == 0) ? '\0' : input_string[strlen(input_string) - 1];

    i = 0;
    argv[i] = strtok(input_string, " ");
    while (argv[i] != NULL) {
        i++;
        argv[i] = strtok(NULL, " ");
    }
}


void exec_command(char **args, char **redir_argv, int wait, int res) {
    for (int i = 0; i < simple_shell_num_builtins(); i++) {
        if (strcmp(args[0], builtin_str[i]) == 0) {
            (*builtin_func[i])(args);
            res = 1;
        }
    }

    if (res == 0) {
        pid_t pid = fork();
        if (pid == 0) {
            if (res == 0) res = simple_shell_redirect(args, redir_argv);
            if (res == 0) res = simple_shell_pipe(args);
            if (res == 0) execvp(args[0], args);
            exit(EXIT_SUCCESS);
        } else if (pid < 0) {
            perror("Error: Fork failed");
            exit(EXIT_FAILURE);
        } else {
            if (wait == 1) waitpid(pid, NULL, 0);
        }
    }
}


int simple_shell_cd(char **args) {
    if (args[1] == NULL) {
        fprintf(stderr, "Error: Expected argument to \"cd\"\n");
    } else {
        if (chdir(args[1]) perror("Error: Failed to change directory");
    }
    return 1;
}

int simple_shell_help(char **args) {
    printf("Built-in commands:\n");
    for (int i = 0; i < simple_shell_num_builtins(); i++) {
        printf("  %s\n", builtin_str[i]);
    }
    return 1;
}

int simple_shell_exit(char **args) {
    running = 0;
    return running;
}


int simple_shell_history(char *history, char **redir_args) {
    if (strcmp(history, "No commands in history") == 0) {
        fprintf(stderr, "No commands in history\n");
        return 1;
    }
    printf("%s\n", history);
    return 0;
}


void set_prev_command(char *history, char *line) {
    strncpy(history, line, MAX_LINE_LENGTH);
}


int simple_shell_redirect(char **args, char **redir_argv) {
    int redir_op_index = is_redirect(args);
    if (redir_op_index != 0) {
        parse_redirect(args, redir_argv, redir_op_index);
        if (strcmp(redir_argv[0], ">") == 0) {
            exec_child_overwrite_to_file(args, redir_argv);
        } else if (strcmp(redir_argv[0], "<") == 0) {
            exec_child_overwrite_from_file(args, redir_argv);
        } else if (strcmp(redir_argv[0], ">>") == 0) {
            exec_child_append_to_file(args, redir_argv);
        }
        return 1;
    }
    return 0;
}


int simple_shell_pipe(char **args) {
    int pipe_op_index = is_pipe(args);
    if (pipe_op_index != 0) {
        char *child01_arg[PIPE_SIZE];
        char *child02_arg[PIPE_SIZE];
        parse_pipe(args, child01_arg, child02_arg, pipe_op_index);
        exec_child_pipe(child01_arg, child02_arg);
        return 1;
    }
    return 0;
} 
