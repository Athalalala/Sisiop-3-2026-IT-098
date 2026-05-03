# Sisiop-3-2026-IT-098

## Profil Mahasiswa
Nama           : Jude Athala Yazid Sari

NRP            : 5027251098

Departemen     : Teknologi Informasi

Kelas          : Sistem Operasi A
 

## **Laporan Penyelesaian Soal Praktikum Modul 3**
Repository ini berisi *source code* dan penjelasan mengenai penyelesaian soal Praktikum Modul 3 untuk mata kuliah Sistem Operasi. Modul ini berfokus pada *navi & wired*pada sistem operasi.

### Problem Keseluruhan
Pada penyelesaian Soal Modul 3 ini saya juga menggunakan bantuan chat gpt karena di peraturan diperbolehkan menggunakan Ai. Disini saya menggunakan chat gpt untuk membuat script, menjelaskan alur dan memecahkan masalah pada penyelesaian soal Modul 3. Berikut link percakapan saya dengan chat gpt
[Chat gpt](https://chatgpt.com/share/69f17ab6-9104-8322-81bf-6a889c597af0)

### Berikut langkah - langkah pengerjaan soal_1

#### langkah pertama

**inisiasi**

pertama-tama saat,kita masuk ke direktori praktikum, karena saat mengerjakan sudah ada direktori soal_1, kita buat direktori modul_3 `mkdir modul_3` lalu masuk ke modul_3 `cd modul_3`. selanjutnya buat direktori soal_1 `mkdir soal_1`, lalu masuk soal_1 `cd soal_1'.

### langkah kedua

buat file header 'nano protokol.h'

lalu isi dengan

```awk
#ifndef PROTOCOL_H
#define PROTOCOL_H

#define PORT 8080
#define MAX_CLIENTS 100
#define BUFFER_SIZE 1024

typedef struct {
    char sender[50];
    char content[BUFFER_SIZE];
} Message;

#endif
```

### langkah ketiga 

membuat file program navi.c untuk bagian pengguna dan wired.c sebagai server

`nano navi.c` lalu diisi dengan code seperti ini:

```awk
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include "protocol.h"

int main() {
    int sock;
    struct sockaddr_in server;
    char buffer[BUFFER_SIZE], name[50];

    sock = socket(AF_INET, SOCK_STREAM, 0);
    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);

    if (connect(sock, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("Connect failed"); return 1;
    }

    printf("Enter your name: ");
    fgets(name, sizeof(name), stdin);
    name[strcspn(name, "\n")] = '\0';
    send(sock, name, strlen(name), 0);

    fd_set readfds;
    while (1) {
        FD_ZERO(&readfds);
        FD_SET(0, &readfds);
        FD_SET(sock, &readfds);

        if (select(sock + 1, &readfds, NULL, NULL, NULL) < 0) continue;

        if (FD_ISSET(sock, &readfds)) {
            int val = recv(sock, buffer, BUFFER_SIZE - 1, 0);
            if (val <= 0) { printf("\nDisconnected from The Wired.\n"); break; }
            buffer[val] = '\0';
            printf("%s", buffer);
            fflush(stdout);
            
            // Auto-exit jika ditolak server
            if (strstr(buffer, "Name already used") || strstr(buffer, "Wrong password")) break;
        }

        if (FD_ISSET(0, &readfds)) {
            fgets(buffer, BUFFER_SIZE, stdin);
            send(sock, buffer, strlen(buffer), 0);
            if (strncmp(buffer, "/exit", 5) == 0) break;
        }
    }
    close(sock);
    return 0;
}
```

dan file program 

`nano wired.c`

lalu diisi dengan code seperti berikut 
```awk
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include <time.h>
#include "protocol.h"

typedef struct {
    int fd;
    char name[50];
    int is_admin;
} Client;

Client clients[MAX_CLIENTS];
time_t start_time;

void log_event(const char *category, const char *msg) {
    FILE *f = fopen("history.log", "a");
    if (!f) return;
    time_t t = time(NULL);
    struct tm tm = *localtime(&t);
    fprintf(f, "[%04d-%02d-%02d %02d:%02d:%02d] [%s] [%s]\n",
            tm.tm_year + 1900, tm.tm_mon + 1, tm.tm_mday,
            tm.tm_hour, tm.tm_min, tm.tm_sec, category, msg);
    fclose(f);
}

void broadcast(const char *msg, int sender_fd) {
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i].fd > 0 && clients[i].fd != sender_fd && !clients[i].is_admin) {
            send(clients[i].fd, msg, strlen(msg), 0);
        }
    }
}

int main() {
    int server_fd, max_fd, new_socket;
    struct sockaddr_in address;
    fd_set readfds;
    start_time = time(NULL);

    for (int i = 0; i < MAX_CLIENTS; i++) clients[i].fd = 0;

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("Bind failed"); exit(1);
    }
    listen(server_fd, 10);

    printf("Server started on port %d...\n", PORT);
    log_event("System", "SERVER ONLINE");

    while (1) {
        FD_ZERO(&readfds);
        FD_SET(server_fd, &readfds);
        max_fd = server_fd;

        for (int i = 0; i < MAX_CLIENTS; i++) {
            if (clients[i].fd > 0) {
                FD_SET(clients[i].fd, &readfds);
                if (clients[i].fd > max_fd) max_fd = clients[i].fd;
            }
        }

        if (select(max_fd + 1, &readfds, NULL, NULL, NULL) < 0) continue;

        if (FD_ISSET(server_fd, &readfds)) {
            new_socket = accept(server_fd, NULL, NULL);
            char temp_name[50];
            int len = recv(new_socket, temp_name, sizeof(temp_name) - 1, 0);
            if (len <= 0) { close(new_socket); continue; }
            temp_name[len] = '\0';
            temp_name[strcspn(temp_name, "\r\n")] = '\0';

            // AUTH ADMIN
            int is_knights = (strcmp(temp_name, "The Knights") == 0);
            if (is_knights) {
                send(new_socket, "Enter Password: ", 16, 0);
                char pass[50];
                int plen = recv(new_socket, pass, sizeof(pass) - 1, 0);
                if (plen <= 0) { close(new_socket); continue; }
                pass[plen] = '\0';
                pass[strcspn(pass, "\r\n")] = '\0';

                if (strcmp(pass, "protocol7") != 0) {
                    send(new_socket, "[System] Wrong password\n", 24, 0);
                    close(new_socket); continue;
                }
                send(new_socket, "[System] Authentication Successful. Granted Admin privileges.\n", 62, 0);
            }

            // DUPLICATE NAME CHECK
            int exists = 0;
            for (int i = 0; i < MAX_CLIENTS; i++) {
                if (clients[i].fd > 0 && strcmp(clients[i].name, temp_name) == 0) { exists = 1; break; }
            }
            if (exists) {
                send(new_socket, "[System] Name already used\n", 27, 0);
                close(new_socket); continue;
            }

            // REGISTER CLIENT
            for (int i = 0; i < MAX_CLIENTS; i++) {
                if (clients[i].fd == 0) {
                    clients[i].fd = new_socket;
                    strcpy(clients[i].name, temp_name);
                    clients[i].is_admin = is_knights;
                    
                    char logmsg[100];
                    sprintf(logmsg, "User '%s' connected", temp_name);
                    log_event("System", logmsg);

                    char welcome[100];
                    sprintf(welcome, "--- Welcome to The Wired, %s ---\n", temp_name);
                    send(new_socket, welcome, strlen(welcome), 0);

                    if (is_knights) {
                        char *menu = "\n=== THE KNIGHTS CONSOLE ===\n1. Check Active Entities\n2. Check Server Uptime\n3. Execute Emergency Shutdown\nCommand >> ";
                        send(new_socket, menu, strlen(menu), 0);
                    }
                    break;
                }
            }
        }

        for (int i = 0; i < MAX_CLIENTS; i++) {
            if (clients[i].fd > 0 && FD_ISSET(clients[i].fd, &readfds)) {
                char buf[BUFFER_SIZE];
                int val = recv(clients[i].fd, buf, BUFFER_SIZE - 1, 0);
                if (val <= 0 || strncmp(buf, "/exit", 5) == 0) {
                    char logmsg[100];
                    sprintf(logmsg, "User '%s' disconnected", clients[i].name);
                    log_event("System", logmsg);
                    close(clients[i].fd);
                    clients[i].fd = 0;
                    clients[i].name[0] = '\0';
                } else {
                    buf[val] = '\0';
                    buf[strcspn(buf, "\r\n")] = '\0';

                    if (clients[i].is_admin) {
                        if (strcmp(buf, "1") == 0) {
                            int count = 0;
                            for(int j=0; j<MAX_CLIENTS; j++) if(clients[j].fd > 0 && !clients[j].is_admin) count++;
                            char res[100]; sprintf(res, "[System] Active Entities: %d\nCommand >> ", count);
                            send(clients[i].fd, res, strlen(res), 0);
                            log_event("Admin", "RPC_GET_USERS");
                        } else if (strcmp(buf, "2") == 0) {
                            char res[100]; sprintf(res, "[System] Uptime: %ld sec\nCommand >> ", time(NULL) - start_time);
                            send(clients[i].fd, res, strlen(res), 0);
                            log_event("Admin", "RPC_GET_UPTIME");
                        } else if (strcmp(buf, "3") == 0) {
                            log_event("Admin", "RPC_SHUTDOWN");
                            log_event("System", "EMERGENCY SHUTDOWN INITIATED");
                            broadcast("[System] Server shutting down...\n", -1);
                            exit(0);
                        }
                    } else {
                        char out[BUFFER_SIZE + 60];
                        sprintf(out, "[%s]: %s\n", clients[i].name, buf);
                        broadcast(out, clients[i].fd);
                        char logmsg[BUFFER_SIZE + 70]; sprintf(logmsg, "[%s]: %s", clients[i].name, buf);
                        log_event("User", logmsg);
                    }
                }
            }
        }
    }
}
```
### langkah ke 4

kita compile program yang telah dibuat sebelumnya dengan cara 

`gcc wired.c -o wired`

`gcc navi.c -o navi`

### langkah ke 5

jalankan `wired.c` untuk di terminal 1

lalu buat terminal baru sebanyak 2 tab dan jalankan `./navi` dan jangan lupa untuk masuk ke direktori yang dituju dengan cara

`cd praktikum/modul_3/soal_1`

### langkah ke 6 

`cat history.log` untuk cek log yang sudah ada sebelumnya.

### expected output 
1.Koneksi via IP + PORT dari protocol.h

2.Client asinkron (pakai select(), bukan fork)

3.Server bisa handle banyak client (non-blocking)

4.Nama harus unik (ini penting banget)

5.Broadcast ke semua client

6.Admin system + password + menu

7.Logging format khusus (history.log)

![image link](Assets/gambar_67.png)

![image link](Assets/gambar_68.png)

![image link](Assets/gambar_69.png)


