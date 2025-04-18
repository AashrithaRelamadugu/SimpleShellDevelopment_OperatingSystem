# SimpleShellDevelopment_OperatingSystem
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <pwd.h>
#include <errno.h>

#define MAX_CMD_LEN 1024
#define MAX_ARGS 100
#define MAX_CMDS 20
#define HISTORY_FILE ".my_shell_history"

int command_count = 1;

void save_history(const char *cmd) {
    char path[256];
    snprintf(path, sizeof(path), "%s/%s", getenv("HOME"), HISTORY_FILE);
    FILE *fp = fopen(path, "a");
    if (fp) {
        fprintf(fp, "%s\n", cmd);
        fclose(fp);
    }
}

void show_history() {
    char path[256];
    snprintf(path, sizeof(path), "%s/%s", getenv("HOME"), HISTORY_FILE);
    FILE *fp = fopen(path, "r");
    if (!fp) {
        perror("history");
        return;
    }
    char line[1024];
    int num = 1;
    while (fgets(line, sizeof(line), fp)) {
        printf("%d  %s", num++, line);
    }
    fclose(fp);
}

void parse_command(char *cmd, char **args) {
    int i = 0;
    args[i] = strtok(cmd, " \t\r\n");
    while (args[i] != NULL && i < MAX_ARGS - 1) {
        args[++i] = strtok(NULL, " \t\r\n");
    }
    args[i] = NULL;
}

void execute_pipeline(char *line) {
    char *commands[MAX_CMDS];
    int num_cmds = 0;

    commands[num_cmds] = strtok(line, "|");
    while (commands[num_cmds] != NULL && num_cmds < MAX_CMDS - 1) {
        commands[++num_cmds] = strtok(NULL, "|");
    }

    int pipefd[2 * (num_cmds - 1)];
    for (int i = 0; i < num_cmds - 1; ++i) {
        if (pipe(pipefd + i * 2) < 0) {
            perror("pipe");
            exit(EXIT_FAILURE);
        }
    }

    for (int i = 0; i < num_cmds; ++i) {
        char *args[MAX_ARGS];
        parse_command(commands[i], args);

        pid_t pid = fork();
        if (pid == 0) {
            if (i > 0)
                dup2(pipefd[(i - 1) * 2], STDIN_FILENO);
            if (i < num_cmds - 1)
                dup2(pipefd[i * 2 + 1], STDOUT_FILENO);

            for (int j = 0; j < 2 * (num_cmds - 1); ++j)
                close(pipefd[j]);

            execvp(args[0], args);
            perror("execvp");
            exit(1);
        }
    }

    for (int i = 0; i < 2 * (num_cmds - 1); ++i)
        close(pipefd[i]);
    for (int i = 0; i < num_cmds; ++i)
        wait(NULL);
}

void execute_command(char *line) {
    char *args[MAX_ARGS];
    char *input_file = NULL, *output_file = NULL;

    char *in = strchr(line, '<');
    char *out = strchr(line, '>');

    if (in) {
        *in = '\0';
        input_file = strtok(in + 1, " \t\r\n");
    }
    if (out) {
        *out = '\0';
        output_file = strtok(out + 1, " \t\r\n");
    }

    parse_command(line, args);
    if (args[0] == NULL) return;

    pid_t pid = fork();
    if (pid == 0) {
        if (input_file) {
            int fd = open(input_file, O_RDONLY);
            if (fd < 0) { perror("input"); exit(1); }
            dup2(fd, STDIN_FILENO);
            close(fd);
        }

        if (output_file) {
            int fd = open(output_file, O_WRONLY | O_CREAT | O_TRUNC, 0644);
            if (fd < 0) { perror("output"); exit(1); }
            dup2(fd, STDOUT_FILENO);
            close(fd);
        }

        execvp(args[0], args);
        perror("execvp");
        exit(1);
    } else {
        wait(NULL);
    }
}

int run_command(char *line) {
    int status = 0;

    // Handle && and ||
    if (strstr(line, "&&") || strstr(line, "||")) {
        char *op = strstr(line, "&&") ? "&&" : "||";
        char *cmd1 = strtok(line, op);
        char *cmd2 = strtok(NULL, "");

        if (!cmd1 || !cmd2) return 1;

        status = run_command(cmd1);
        if ((strstr(op, "&&") && status == 0) || (strstr(op, "||") && status != 0)) {
            return run_command(cmd2);
        }
        return status;
    }

    // Background execution
    int bg = 0;
    int len = strlen(line);
    if (len > 0 && line[len - 1] == '&') {
        bg = 1;
        line[len - 1] = '\0';
    }

    // Built-in cd
    if (strncmp(line, "cd ", 3) == 0) {
        char *path = line + 3;
        if (chdir(path) != 0) {
            perror("cd failed");
            return 1;
        }
        return 0;
    }

    // Built-in history
    if (strcmp(line, "history") == 0) {
        show_history();
        return 0;
    }

    if (strchr(line, '|')) {
        execute_pipeline(line);
    } else {
        execute_command(line);
    }

    return bg ? 0 : 0;
}

int main() {
    char line[MAX_CMD_LEN];

    while (1) {
        // Get current directory and user
        char cwd[256];
        getcwd(cwd, sizeof(cwd));
        struct passwd *pw = getpwuid(getuid());
        char *user = pw->pw_name;

        // Display prompt with count
        printf("💻 [%d] %s:%s$ ", command_count++, user, cwd);
        fflush(stdout);

        if (!fgets(line, sizeof(line), stdin)) break;
        line[strcspn(line, "\n")] = 0;

        if (strlen(line) == 0) continue;
        if (strcmp(line, "exit") == 0) break;

        save_history(line);
        run_command(line);
    }

    return 0;
}
