# Soal 1
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
