# Perbaikan: nginx:alpine polos tidak punya 'netstat', padahal Tugas 6
# (B3 - SYN flood) butuh netstat untuk menghitung SYN_RECV di sisi target.
# Image ini menambahkan net-tools di atas nginx:alpine resmi.
FROM nginx:alpine
RUN apk add --no-cache net-tools
