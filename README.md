# cepheus-kernel-builder — BBRv2 patch set

Patch + workflow eksperimen untuk menambahkan **TCP BBRv2** ke kernel
`4.14.357-openela-perf` di Xiaomi Mi 9 (`cepheus`, Android 17) memakai
source kernel dan build script milik sendiri.

## Sumber yang dipakai

- Kernel source: `indramccaslin/android_kernel_xiaomi_sm8150` branch `17-dev`
  - versi `4.14.357`
  - sudah punya `net/sched/sch_fq.c`
  - sudah punya `net/ipv4/tcp_bbr.c` (BBR v1)
  - sudah ada KernelSU-Next + SUSFS in-tree
- Config + workflow referensi: `indramccaslin/cepheus-kernel-builder` branch `main`
  - `configs/cepheus-4.1.config` = config kernel yang sedang terpasang
- Patch backport BBRv2 4.14: `zesakain/BBRv2-Kernel-4.14-Rosemary` branch `vEleatic-BBR2`

Patch rosemary diadaptasi: patch yang kotor (`Kconfig.txt`, `tcp_plb.o`,
`tcp_bbrplus`, paksa `tcp_cong.c`, dan SNMP `TCPECNRehash`) dibuang dan
diganti dengan patch Kconfig/Makefile yang bersih.

## Isi direktori

```text
patches/
  0001-bbr2-tcp_bbr2.c.patch          # tcp_bbr2.c dari backport 5.13 -> 4.14
  0002-bbr2-tcp_dctcp.h.patch         # helper DCTCP ECN yang dipakai bbr2
  0003-bbr2-net-tcp.h-fields.patch    # field rate_sample / tcp_congestion_ops / tcp_skb_cb
  0004-bbr2-linux-tcp.h-fields.patch  # field struct tcp_sock (tcp_clock_cache, dll)
  0005-bbr2-reord-seen.patch          # field reord_seen
  0006-bbr2-inet_diag.h.patch         # struct tcp_bbr2_info untuk ss -tin
  0007-bbr2-kconfig.patch             # CONFIG_TCP_CONG_BBR2 + DEFAULT_BBR2
  0008-bbr2-makefile.patch            # obj-$(CONFIG_TCP_CONG_BBR2) += tcp_bbr2.o
configs/
  cepheus-4.1-bbr2.config             # config dengan BBR2 + FQ aktif
.github/workflows/
  build-bbr2.yml                      # workflow GitHub Actions
```

## Cara pakai

1. Copy isi folder ini ke repo `cepheus-kernel-builder` kamu.
2. Commit & push.
3. Buka **Actions → Build cepheus kernel 4.14.357 + BBRv2 → Run workflow**.
4. `kernel_ref` biarkan `17-dev`.
5. Setelah selesai, download artifact `cepheus-bbr2-kernel`, ekstrak
   `cepheus-bbr2-*.zip`, flash lewat fastboot/AnyKernel3:
   ```sh
   fastboot flash boot boot.img   # atau flash zip di recovery
   fastboot reboot
   ```
6. Setelah boot, aktifkan:
   ```sh
   su -c 'sysctl -w net.core.default_qdisc=fq'
   su -c 'sysctl -w net.ipv4.tcp_congestion_control=bbr2'
   ```
7. Cek:
   ```sh
   su -c 'cat /proc/sys/net/ipv4/tcp_available_congestion_control'
   su -c 'cat /proc/sys/net/ipv4/tcp_congestion_control'
   su -c 'tc qdisc show dev wlan0 | head'
   ```

Harusnya muncul `bbr2` di `tcp_available_congestion_control` dan qdisc
`fq` sebagai leaf dari `mq`.

## Persisten

Taruh ini di script init KernelSU (mis. `/data/adb/service.d/99-bbr2.sh`):

```sh
#!/system/bin/sh
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr2
```

Jangan lupa `chmod 755`.

## ⚠️ Catatan penting

- Ini **backport eksperimen**, bukan BBRv2 upstream. BBRv2 belum masuk mainline.
- Patch rosemary hanya menyontek sebagian struktur dari kernel 5.13.
  Beberapa tempat seperti `tcp_rate.c`, `tcp_output.c`, dan `tcp_input.c`
  tidak ikut di-backport, jadi field seperti `delivered_ce`, `tx_in_flight`,
  `lost`, dan `is_ack_delayed` mungkin tidak terisi sempurna di kernel 4.14.
- ECN/DCTCP path perlu diuji. Kalau mengalami packet loss/retransmisi
  tinggi, kembali ke `cubic`:
  ```sh
  su -c 'sysctl -w net.ipv4.tcp_congestion_control=cubic'
  ```
- Backup `boot.img` sebelum flash. Kalau bootloop, flash kembali boot.img lama.
- Kalau build gagal karena `patch` conflict, cek log workflow. Kemungkinan
  besar konflik di `include/linux/tcp.h` atau `include/net/tcp.h`;
  sesuaikan hunk dengan source branch `17-dev` yang terbaru.
