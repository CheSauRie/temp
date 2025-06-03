# temp
import subprocess
import threading
import time
from datetime import datetime
import re
import argparse

LOG_TIME_FORMAT = "%Y/%m/%d %H:%M:%S"

# Shared list to hold timestamps when ping failed
ping_fail_times = []

def run_ping(host, duration=60):
    print(f"[*] Bắt đầu ping tới {host} trong {duration} giây...")
    end_time = time.time() + duration
    while time.time() < end_time:
        try:
            response = subprocess.run(["ping", "-c", "1", "-W", "1", host],
                                      stdout=subprocess.PIPE,
                                      stderr=subprocess.PIPE)
            if response.returncode != 0:
                # Ping failed
                ping_fail_times.append(datetime.now())
        except Exception as e:
            print(f"[!] Lỗi khi ping: {e}")
        time.sleep(1)
    print(f"[✓] Kết thúc ping tới {host}")

def parse_log_time(line):
    match = re.match(r'^(\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2})', line)
    if match:
        return datetime.strptime(match.group(1), LOG_TIME_FORMAT)
    return None

def analyze_webrtc_reconnect(log_path):
    last_disconnect_time = None
    results = []

    with open(log_path, 'r', encoding='utf-8') as f:
        for line in f:
            if '[webrtc]' not in line:
                continue

            timestamp = parse_log_time(line)

            if 'peer connection closed' in line:
                last_disconnect_time = timestamp

            elif 'peer connection established' in line and last_disconnect_time:
                delta = (timestamp - last_disconnect_time).total_seconds()
                ping_fail_during = [p for p in ping_fail_times if last_disconnect_time <= p <= timestamp]

                results.append({
                    'disconnected_at': last_disconnect_time,
                    'reconnected_at': timestamp,
                    'downtime_seconds': delta,
                    'ping_failures': ping_fail_during
                })
                last_disconnect_time = None

    # Print result
    for item in results:
        print(f"\n[!] Mất kết nối WebRTC lúc: {item['disconnected_at']}")
        print(f"[+] Kết nối lại lúc: {item['reconnected_at']}")
        print(f"⏱️  Downtime: {item['downtime_seconds']} giây")
        if item['ping_failures']:
            print(f"[x] Trong thời gian này mất {len(item['ping_failures'])} gói ping:")
            for pf in item['ping_failures']:
                print(f"    ⚠️ Ping thất bại lúc {pf}")
        else:
            print("[✓] Không mất ping — khả năng do WebRTC bị drop riêng.")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Phân tích downtime WebRTC và kiểm tra ping song song")
    parser.add_argument("log_file", help="Đường dẫn tới file log MediaMTX")
    parser.add_argument("ping_host", help="Địa chỉ IP hoặc domain server cần ping")
    parser.add_argument("--ping_duration", type=int, default=60, help="Thời gian chạy ping song song (giây)")

    args = parser.parse_args()

    # Chạy ping ở luồng phụ
    ping_thread = threading.Thread(target=run_ping, args=(args.ping_host, args.ping_duration))
    ping_thread.start()

    # Trong lúc ping, chờ xong rồi phân tích log
    ping_thread.join()
    analyze_webrtc_reconnect(args.log_file)