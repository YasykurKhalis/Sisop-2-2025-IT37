# Soal 1
a. Downloading the Clues
```c
void download_and_unzip() {
    DIR* dir = opendir("Clues");
    if (dir) {
        closedir(dir);
        return;
    }

    pid_t pid = fork();
    if (pid == 0) {
        char *args[] = {"wget", "-q", "https://drive.google.com/uc?export=download&id=1xFn1OBJUuSdnApDseEczKhtNzyGekauK", "-O", "Clues.zip", NULL};
        execvp("wget", args);
        perror("execvp wget");
        exit(EXIT_FAILURE);
    } else {
        wait(NULL);
    }

    pid = fork();
    if (pid == 0) {
        char *args[] = {"unzip", "-q", "Clues.zip", NULL};
        execvp("unzip", args);
        perror("execvp unzip");
        exit(EXIT_FAILURE);
    } else {
        wait(NULL);
    }

    remove("Clues.zip");
}
```
Penjelasan :

Function ini berfungsi untuk mendownload zip dari link yg telah disediakan, kemudian zip tersebut akan di unzip dan dimasukkan ke dalam sebuah folder, dan fucntion ini tidak akan mendownload kembali file zipnya jika file zip tersebut sudah didownload
![image](https://github.com/user-attachments/assets/cd7a993e-580f-4c31-b2b5-0e429c31ef0e)
![image](https://github.com/user-attachments/assets/cd8e9e65-2d2a-49d9-a745-c01679929cce)

b. Filtering the Files
```c
bool is_valid_filename(const char *filename) {
    if (strlen(filename) != 5) return false;
    return (isalpha(filename[0]) || isdigit(filename[0])) && (strcmp(&filename[1], ".txt") == 0);
}

void filter_files() {
    mkdir("Filtered", 0755);

    struct dirent *entry;
    DIR *dp;

    for (int i = 0; i < 4; i++) {
        char dirname[20];
        sprintf(dirname, "Clues/Clue%c", 'A' + i);

        dp = opendir(dirname);
        if (dp == NULL) {
            perror("opendir");
            continue;
        }

        while ((entry = readdir(dp)) != NULL) {
            if (entry->d_type == DT_REG && is_valid_filename(entry->d_name)) {
                char old_path[256], new_path[256];
                sprintf(old_path, "%s/%s", dirname, entry->d_name);
                sprintf(new_path, "Filtered/%s", entry->d_name);

                if (rename(old_path, new_path) != 0) {
                    perror("rename");
                }
            } else if (entry->d_type == DT_REG && strstr(entry->d_name, ".txt") != NULL) {
                char filepath[256];
                sprintf(filepath, "%s/%s", dirname, entry->d_name);
                if (remove(filepath) != 0) {
                    perror("remove");
                }
            }
        }
        closedir(dp);
    }
}
```
Penjelasan :

Function ini akan masuk ke setiap folder yg sudah di unzip kemudian akan mengecek nama filenya apakah sesuai ketentuan atau tidak, jika namanya sesuai maka akan dimasukkan ke dalam folder lain dan jika salah maka akan langsung dihapus sehingga ketika semua sudah di filter maka folder sebelumnya akan kosong

![image](https://github.com/user-attachments/assets/ddc5a362-1f5b-48e0-8552-36b9b0584e5d)
![image](https://github.com/user-attachments/assets/02fb8c66-6337-4a7c-9d34-7edca50a2526)

c. Combine the File Content
```c
int compare_files(const void *a, const void *b) {
    const char *file1 = *(const char **)a;
    const char *file2 = *(const char **)b;

    if (isdigit(file1[0]) && !isdigit(file2[0])) return -1;
    if (!isdigit(file1[0]) && isdigit(file2[0])) return 1;

    if (isdigit(file1[0]) && isdigit(file2[0])) {
        return file1[0] - file2[0];
    }

    return file1[0] - file2[0];
}

void combine_files() {
    FILE *combined = fopen("Combined.txt", "w");
    if (!combined) {
        perror("Error creating Combined.txt");
        return;
    }

    // Urutan file: 1.txt, a.txt, 2.txt, b.txt, dst.
    const char *files[] = {"1.txt", "a.txt", "2.txt", "b.txt", "3.txt", "c.txt", "4.txt", "d.txt", "5.txt", "e.txt", "6.txt", "f.txt"};
    int num_files = 12;

    for (int i = 0; i < num_files; i++) {
        char filepath[256];
        sprintf(filepath, "Filtered/%s", files[i]);

        FILE *file = fopen(filepath, "r");
        if (file) {
            char ch;
            if (fread(&ch, 1, 1, file) == 1) {
                fputc(ch, combined);  // Tulis karakter ke Combined.txt
            }
            fclose(file);
        }
    }

    fclose(combined);
}
```
Penjelasan :

Function ini akan mengcompare terlebih dahulu filenya apakah benar angka dan digit sehingga saat akan di combine akan sesuai urutan yaitu angka terkecil terlebih dahulu lalu huruf lalu angka lagi dan seterusnya, kemudian hasilnya akan di masukkan ke dalam file txt dan semua file dari folder sebelumnya akan dihapus

![image](https://github.com/user-attachments/assets/e4fa3972-ae26-4de4-b71e-435897b5968c)
![image](https://github.com/user-attachments/assets/124b97bf-dd07-49e0-a4f9-fd310cc58701)

d. Decode the file
```c
void rot13(char *str) {
    for (int i = 0; str[i]; i++) {
        if (isalpha(str[i])) {
            if ((tolower(str[i]) - 'a') < 13) {
                str[i] += 13;
            } else {
                str[i] -= 13;
            }
        }
    }
}

void decode_file() {
    FILE *combined = fopen("Combined.txt", "r");
    if (combined == NULL) {
        perror("Error opening Combined.txt");
        return;
    }

    char content[MAX_CONTENT];
    if (!fgets(content, MAX_CONTENT, combined)) {
        fclose(combined);
        perror("Error reading Combined.txt");
        return;
    }
    fclose(combined);

    rot13(content);

    FILE *decoded = fopen("Decoded.txt", "w");
    if (decoded == NULL) {
        perror("Error creating Decoded.txt");
        return;
    }

    fputs(content, decoded);
    fclose(decoded);
}
```
Penjelasan :

Function ini akan menggunakan Rot13 untuk decode string dari file txt yg sebelumnya berisi hasil dari combined txt yg sudah kita dapatkan, hasil nya akan di decode dan dimasukkan ke file txt yg lain

![image](https://github.com/user-attachments/assets/94dc426f-ae93-4b60-aefe-a5c9d401c13e)
![image](https://github.com/user-attachments/assets/db33358a-1bb9-4525-8f17-069d36aac916)

e. Password Check
```
BewareOfAmpy
```
Penjelasan :

Ini adalah hasil yg didapatkan ketika membuka file txt decoded.txt dan ketika dicoba hasilnya di web yg ditentukan ternyata benar
![image](https://github.com/user-attachments/assets/0bd25afb-19e7-4497-8a8b-579403c6fa5a)

# Soal 2
Di soal 2 kita diminta untuk membuat daemon yang bisa mendekripsi nama file.

a. Mengunduh file zip dan mengekstraknya
```c
void downloadZipFile() {
    char *file_id = "1_5GxIGfQr3mNKuavJbte_AoRkEQLXSKS";
    char *filename = "starterkit.zip";

    struct stat st;

    if (stat(FOLDER_STARTERKIT, &st) == -1) {
        mkdir(FOLDER_STARTERKIT, 0777);
    }

    char command[512]; // inisiasi buffer
    snprintf(command, sizeof(command),
    "wget --no-check-certificate \"https://docs.google.com/uc?export=download&id=%s\" -O %s",
    file_id, filename); // membuat string dan disimpan ke buffer

    system(command);
    system("unzip starterkit.zip -d starter_kit");
    system("rm starterkit.zip");
}
```
Penjelasan:

Function ini berfungsi untuk membuat folder tempat menampung file zip yang sudah diekstrak sekaligus mengunduh file zip sesuai dari tautan yang sudah diberikan dengan menggunakan command linux yang dieksekusi menggunakan `system()`.

Isi folder starter_kit:

![Capture](https://github.com/user-attachments/assets/2c04585d-de6f-4b85-969c-937248a6edbb)

b. Membuat directory karantina yang dan mendekripsi nama file
```c
void decryptFileName() { // soal 2
    pid_t pid = fork();
    if (pid > 0) exit(0); // Parent keluar
    if (pid < 0) exit(1); // Fork gagal

    setsid();

    // Log
    char logbuf[128];
    snprintf(logbuf, sizeof(logbuf), "Successfully started decryption process with PID %d.", getpid());
    writeLog(logbuf);

    // dapatkan direktori saat ini
    char cwd[PATH_MAX];
    getcwd(cwd, sizeof(cwd));
    chdir(cwd); 

    struct stat st;

    if (stat(FOLDER_QUARANTINE, &st) == -1) {
        mkdir(FOLDER_QUARANTINE, 0777);
    }

    // menutup input/output/error terminal
    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);

    while (1) { // berjalan terus (daemon)
        DIR *folder = opendir(FOLDER_QUARANTINE);
        struct dirent *entry;

        if (folder != NULL) {
            while ((entry = readdir(folder)) != NULL) { // iterasi semua file 
                if (entry->d_type == DT_REG) { // hanya memproses file reguler
                    char oldpath[512], newname[256], newpath[512];
                    snprintf(oldpath, sizeof(oldpath), "%s/%s", FOLDER_QUARANTINE, entry->d_name);

                    // Cek apakah nama ada ekstensinya jika tidak ada di lewati
                    size_t len = strlen(entry->d_name);
                    if (strrchr(entry->d_name, '.') != NULL) continue;

                    char command[512];
                    snprintf(command, sizeof(command), "echo %s | base64 -d", entry->d_name); // decrypt nama file dengan base64

                    FILE *fp = popen(command, "r"); // jalankan command base64 decoding dan ambil outputnya
                    if (fp == NULL) continue;

                    // jika gagal mendapatkan nama yang sudah didecrypt maka akan diabaikan
                    if (fgets(newname, sizeof(newname), fp) == NULL) {
                        pclose(fp);
                        continue;
                    }

                    pclose(fp);
                    newname[strcspn(newname, "\n")] = 0; // menghapus newline

                    snprintf(newpath, sizeof(newpath), "%s/%s", FOLDER_QUARANTINE, newname); // ubah nama file
                    rename(oldpath, newpath);
                }
            }
            closedir(folder);
        }
        sleep(5);
    }
}
```
Penjelasan:

Proses ini berjalan sebagai daemon yang berjalan terus-menerus di background dengan menggunakan `while(1)` File yang tidak memiliki ekstensi dianggap terenkripsi dan namanya didekode dari base64 ke nama aslinya dengan menggunakan command `echo "nama_file" | base64 -d`. Setelah didekode, nama file akan diganti menggunakan hasil dekripsi tersebut `rename(oldpath, newpath)`.

`ps aux` untuk daemon dekripsi:

![Capture1](https://github.com/user-attachments/assets/b1d44c72-4188-43c4-a066-0f66d1cfff76)

c. Membuat fitur untuk memindahkan semua file dari folder starter_kit ke quarantine dan sebaliknya
```c
void quarantineFiles() { // soal 3
    DIR *folder = opendir(FOLDER_STARTERKIT);
    struct dirent *entry;

    if (folder != NULL) {
        while ((entry = readdir(folder)) != NULL) {
            if (entry->d_type == DT_REG) {
                char oldpath[512], encoded[256], newpath[512];

                snprintf(oldpath, sizeof(oldpath), "%s/%s", FOLDER_STARTERKIT, entry->d_name);
                snprintf(newpath, sizeof(newpath), "%s/%s", FOLDER_QUARANTINE, entry->d_name);

                rename(oldpath, newpath);

                //Log
                char logbuf[512];
                snprintf(logbuf, sizeof(logbuf), "%s - Successfully moved to quarantine directory.", entry->d_name);
                writeLog(logbuf);
            }
        }
        closedir(folder);
    }
}

void returnFiles() { // soal 3 juga
    DIR *folder = opendir(FOLDER_QUARANTINE);
    struct dirent *entry;

    if (folder != NULL) {
        while ((entry = readdir(folder)) != NULL) {
            if (entry->d_type == DT_REG) {
                char oldpath[512], newname[256], newpath[512];

                snprintf(oldpath, sizeof(oldpath), "%s/%s", FOLDER_QUARANTINE, entry->d_name);
                snprintf(newpath, sizeof(newpath), "%s/%s", FOLDER_STARTERKIT, entry->d_name);

                rename(oldpath, newpath);

                //Log
                char logbuf[512];
                snprintf(logbuf, sizeof(logbuf), "%s - Successfully returned to starter kit directory.", entry->d_name);
                writeLog(logbuf);
            }
        }
        closedir(folder);
    }
}
```
Penjelasan:

Memindahkan semua file dari folder 1 ke folder lain. Pertama masuk ke folder awal (ex:`opendir(FOLDER_STARTERKIT)`), lalu gunakan perulangan while untuk memindahkan semua file dan ganti path file dengan path tujuan. terakhir, tutup folder`closedir(folder)`.

Isi folder quarantine setelah dipindahkan:

![Capture2](https://github.com/user-attachments/assets/b8a6f9a3-89e5-4102-ac69-44539f5d3f69)

d. Tambahkan fitur menghapus file di dalam folder quarantine
```c
void eradicateFiles() { // soal 4
    DIR *folder = opendir(FOLDER_QUARANTINE);
    struct dirent *entry;

    if (folder == NULL) return;

    while ((entry = readdir(folder)) != NULL) {
        if (entry->d_type == DT_REG) {
            char filepath[512];
            snprintf(filepath, sizeof(filepath), "%s/%s", FOLDER_QUARANTINE, entry->d_name);
            remove(filepath); 

            //Log
            char logbuf[512];
            snprintf(logbuf, sizeof(logbuf), "%s - Successfully deleted.", entry->d_name);
            writeLog(logbuf);
        }
    }
    closedir(folder);
}
```
Penjelasan:

Masuk ke dalam folder quarantine dan melakukan perulangan while selama isi folder tidak kosong `while ((entry = readdir(folder)) != NULL) {`. Setelah itu menghapus file dengan `remove(filepath)`.

Isi folder quarantine setelah dihapus isinya(kosong):

![Capture3](https://github.com/user-attachments/assets/0e1cf412-9cd0-48f9-bdcf-ba93785ba08f)

e. Mematikan daemon dekripsi
```c
void shutdownDecrypt() { // soal 5
    FILE *fp = popen("pgrep -af starterkit", "r");
    if (!fp) return;

    char line[256];

    while (fgets(line, sizeof(line), fp)) {
        pid_t pid;
        char cmd[256];

        if (sscanf(line, "%d %[^\n]", &pid, cmd) == 2) { // Pisah PID dan command
            if (strstr(cmd, "--decrypt") && pid != getpid()) { // Spesifikan untuk starterkit --decrypt
                kill(pid, SIGTERM);

                char logbuf[512];
                snprintf(logbuf, sizeof(logbuf), "Successfully shut off decryption process with PID %d.", pid);
                writeLog(logbuf);
            }
        }
    }
    pclose(fp);
}
```
Penjelasan:

Mengambil proses yang sudah kita jalankan `popen("pgrep -af starterkit", "r")` dan menspesifikannya dengan "decrypt" serta mengabaikan proses untuk mematikan daemon dekripsi `strstr(cmd, "--decrypt") && pid != getpid()`. Setelah itu mematikan proses `kill(pid, SIGTERM)`.

`ps aux` setelah proses dimatikan:

![Capture4](https://github.com/user-attachments/assets/366a8401-64db-4b5d-8781-797d639c7ba0)

f. Membuat error handling

Function `int main()`:
```c
int main(int argc, char *argv[]) {

    // Tidak ada argumen, tidak apa-apa
    if (argc == 1) {
        downloadZipFile();
        return 0;
    }

    if (
        strcmp(argv[1], "--decrypt") != 0 &&
        strcmp(argv[1], "--quarantine") != 0 &&
        strcmp(argv[1], "--return") != 0 &&
        strcmp(argv[1], "--eradicate") != 0 &&
        strcmp(argv[1], "--shutdown") != 0
    ) {
        fprintf(stderr, "Argumen %s tidak dikenal\n", argv[1]);
        fprintf(stderr, "Gunakan argumen: --decrypt, --quarantine, --return, --eradicate, atau --shutdown\n");
        return 1;
    }

    downloadZipFile();
    
    if (strcmp(argv[1], "--decrypt") == 0) {
        decryptFileName();
    } else if (strcmp(argv[1], "--quarantine") == 0) {
        quarantineFiles();
    } else if (strcmp(argv[1], "--return") == 0) {
        returnFiles();
    } else if (strcmp(argv[1], "--eradicate") == 0) {
        eradicateFiles();
    } else if (strcmp(argv[1], "--shutdown") == 0) {
        shutdownDecrypt();
    }

    return 0;
}
```
Penjelasan:

Menggunakan parameter `int argc, char *argv[]` untuk menghitung jumlah argumen dan membaca argumen. Jika tidak ada argumen maka hanya akan menjalankan function `downloadZipFile()` dan jika argumen tidak ada dalam function maka akan me-return argumen tidak dikenal.

![Capture5](https://github.com/user-attachments/assets/695f6cc7-475f-4a3f-a59a-160339970ba3)

g. Mencatat log dari setiap penggunaan program
```c
void writeLog(const char *message) { // log untuk soal terakhir
    FILE *logfile = fopen("activity.log", "a");
    if (!logfile) {
        perror("Gagal membuka log file");  // Debug
        return;
    }

    time_t now = time(NULL); // mengambil waktu saat ini
    struct tm *t = localtime(&now); 

    char timebuf[64];
    strftime(timebuf, sizeof(timebuf), "[%d-%m-%Y][%H:%M:%S]", t); 

    fprintf(logfile, "%s - %s\n", timebuf, message); 
    fclose(logfile);
}
```
Penjelasan: 

Function mempunyai argumen berupa pesan untuk program yang dijalankan. Menggunakan `fopen("activity.log", "a")` untuk membuka dan atau membuat file log jika file belum ada. Lalu mengambil waktu saat ini `localtime(&now)` dan menggabungkannya dengan parameter pesan. Terakhir, menuliskannya ke dalam file `activity.log`.

Isi file `activity.log`:

![Capture6](https://github.com/user-attachments/assets/b15fa14e-071f-4539-8099-00c427f0bdce)

#Soal 4
a. Mengetahui semua aktivitas user
```c
void list_processes(const char *user) {
    printf("Listing processes for user: %s\n", user);

    char command[MAX_LINE];
    snprintf(command, sizeof(command), "ps -u %s -o pid,cmd,%%cpu,%%mem 2>/dev/null", user);

    FILE *ps_output = popen(command, "r");
    if (!ps_output) {
        perror("Failed to execute ps command");
        return;
    }

    printf("PID\tCOMMAND\t\t\tCPU\tMEM\n");
    printf("------------------------------------------------\n");

    char line[MAX_LINE];
    while (fgets(line, sizeof(line), ps_output)) {
        printf("%s", line);
    }

    pclose(ps_output);
    write_log("process_list", "RUNNING");
}
```
Penjelasan :

Function ini berguna untuk mengetahui apa saja yg dijalankan oleh user tersebut dimulai dari PID, command, CPU usage, dan juga memory usage
![image](https://github.com/user-attachments/assets/7072ae64-f169-4fcb-8fc8-9067fa8d5acc)

b. Memasang mata-mata dalam mode daemon
```c
void start_daemon(const char *user) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    }

    if (pid > 0) {
        printf("Debugmon daemon started for user %s (PID: %d)\n", user, pid);
        write_log(DAEMON_IDENTIFIER, "RUNNING");
        exit(EXIT_SUCCESS);
    }

    umask(0);
    setsid();
    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);

    while (1) {
        sleep(30);
        // Check and log user processes periodically
        char command[MAX_LINE];
        snprintf(command, sizeof(command), "ps -u %s -o pid= | wc -l", user);
        FILE *ps_output = popen(command, "r");
        if (ps_output) {
            char count_str[16];
            if (fgets(count_str, sizeof(count_str), ps_output)) {
                write_log("daemon_monitoring", "RUNNING");
            }
            pclose(ps_output);
        }
    }
}
```
Penjelasan :

Function ini akan mengecek apa yang akan dilakukan user 

![image](https://github.com/user-attachments/assets/cb2725ef-e76f-435d-ae4c-01e305981fa0)

c. Menghentikan Pengawasan 
```c
void stop_daemon(const char *user) {
    printf("Stopping debugmon daemon for user: %s\n", user);

    char command[MAX_LINE];
    snprintf(command, sizeof(command), "pgrep -f '%s' 2>/dev/null", DAEMON_IDENTIFIER);

    FILE *pgrep_output = popen(command, "r");
    if (!pgrep_output) {
        perror("Failed to find daemon process");
        return;
    }
```
Penjelasan :

Function ini berguna untuk menghentikan proses pengintaian dari daemon

![image](https://github.com/user-attachments/assets/0e1a2216-31a6-4e22-aa5e-5b5858b56b56)

d. Menggagalkan semua proses user yang sedang berjalan
```c
char pid_str[16];
    int found = 0;

    while (fgets(pid_str, sizeof(pid_str), pgrep_output)) {
        pid_t pid = atoi(pid_str);
        if (kill(pid, SIGTERM) == 0) {
            printf("Successfully stopped daemon (PID: %d)\n", pid);
            write_log(DAEMON_IDENTIFIER, "RUNNING");
            found = 1;
        } else {
            fprintf(stderr, "Failed to kill process %d: %s\n", pid, strerror(errno));
        }
    }

    if (!found) {
        printf("No running daemon found for user %s\n", user);
    }

    pclose(pgrep_output);
}

void fail_processes(const char *user) {
    printf("Failing all processes for user: %s\n", user);

    char command[MAX_LINE];
    snprintf(command, sizeof(command), "ps -u %s -o pid= 2>/dev/null", user);

    FILE *ps_output = popen(command, "r");
    if (!ps_output) {
        perror("Failed to get user processes");
        return;
    }

    char pid_str[16];
    int count = 0;

    while (fgets(pid_str, sizeof(pid_str), ps_output)) {
        pid_t pid = atoi(pid_str);
        if (pid > 1 && kill(pid, SIGSTOP) == 0) {  
            printf("Stopped process: %d\n", pid);
            write_log("process_stop", "FAILED");
            count++;
        }
    }

    pclose(ps_output);
    printf("Total processes stopped: %d\n", count);
```
Penjelasan : 

Ketika function ini dijalankan, maka semua proses akan berhenti dan user pun terlogout atau keluar dari user tersebut dan menuju ke root
![image](https://github.com/user-attachments/assets/9cda6fe8-394e-4009-8a3b-3c070b9add73)

e. Mengizinkan user untuk kembali menjalankan proses
```c
void revert_block(const char *user) {
    printf("Reverting block for user: %s\n", user);

    char command[MAX_LINE];
    snprintf(command, sizeof(command), "ps -u %s -o pid= --state T 2>/dev/null", user);

    FILE *ps_output = popen(command, "r");
    if (!ps_output) {
        perror("Failed to get stopped processes");
        return;
    }

    char pid_str[16];
    int count = 0;

    while (fgets(pid_str, sizeof(pid_str), ps_output)) {
        pid_t pid = atoi(pid_str);
        if (pid > 1 && kill(pid, SIGCONT) == 0) {
            printf("Resumed process: %d\n", pid);
            count++;
        }
    }

    pclose(ps_output);
    printf("Total processes resumed: %d\n", count);

    char limit_cmd[MAX_LINE];
    snprintf(limit_cmd, sizeof(limit_cmd), "sudo prlimit --pid 1 --nproc=unlimited --user %s", user);
    system(limit_cmd);

    write_log("user_unblock", "RUNNING");
}
```
Penjelasan :

Function ini akan mengembalikan user agar bisa menjalankan suatu proses, dan proses lainnya akan kembali running tapi akan ada proses juga yg failed

![image](https://github.com/user-attachments/assets/ee7d3627-08fd-4173-a326-33fd77610a5a)

f. Mencatat ke dalam file log
```c
void write_log(const char *process, const char *status) {
    time_t now;
    time(&now);
    struct tm *tm_info = localtime(&now);

    char timestamp[50];
    strftime(timestamp, sizeof(timestamp), "[%d:%m:%Y]-[%H:%M:%S]", tm_info);

    FILE *log = fopen(LOG_FILE, "a");
    if (log) {
        fprintf(log, "%s_%s_STATUS(%s)\n", timestamp, process, status);
        fclose(log);
    }
}
```
Penjelasan :

Semua proses yang sudah dilakukan akan dicatat ke dalam log file yg nantinya bisa dilihat apa saja proses yg sedang running ataupun failed
![image](https://github.com/user-attachments/assets/b1578ac2-0551-4e9b-aedf-4d7425830ada)
