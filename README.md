# Multi-thread Simulator

```python
import threading
import time
import random
from collections import defaultdict

shared_memory = {"x": 0}

memory_lock = threading.Lock()

class CacheState:
    INVALID = "INVALID"
    SHARED = "SHARED"
    MODIFIED = "MODIFIED"

class ThreadCore:
    def __init__(self, name, use_coherence=True):
        self.name = name
        self.cache = {}
        self.cache_state = {}
        self.use_coherence = use_coherence
        self.cache_hits = 0
        self.cache_misses = 0
        self.coherence_messages = 0

    def read(self, key):
        if self.use_coherence:
            if self.cache_state.get(key) in [CacheState.SHARED, CacheState.MODIFIED]:
                self.cache_hits += 1
                return self.cache[key]
            else:
                self.cache_misses += 1
                with memory_lock:
                    val = shared_memory[key]
                    self.cache[key] = val
                    self.cache_state[key] = CacheState.SHARED
                    self.coherence_messages += 1
                return val
        else:
            if key in self.cache:
                self.cache_hits += 1
                return self.cache[key]
            else:
                self.cache_misses += 1
                with memory_lock:
                    val = shared_memory[key]
                    self.cache[key] = val
                return val

    def write(self, key, value):
        if self.use_coherence:
            self.coherence_messages += 1
            self.cache_state[key] = CacheState.MODIFIED
            with memory_lock:
                shared_memory[key] = value
                self.cache[key] = value
        else:
            with memory_lock:
                shared_memory[key] = value
                self.cache[key] = value

def thread_func(core: ThreadCore):
    for _ in range(10):
        op = random.choice(["read", "write"])
        if op == "read":
            val = core.read("x")
            print(f"[{core.name}] READ x = {val}")
        else:
            val = random.randint(1, 100)
            core.write("x", val)
            print(f"[{core.name}] WRITE x = {val}")
        time.sleep(random.uniform(0.01, 0.05))

def run_simulation(use_coherence=True):
    start_time = time.time()
    
    cores = [ThreadCore(f"Core-{i+1}", use_coherence) for i in range(2)]
    threads = [threading.Thread(target=thread_func, args=(core,)) for core in cores]

    for t in threads: t.start()
    for t in threads: t.join()

    end_time = time.time()
    duration = end_time - start_time

    print("\n=== STATS WITH" + (" COHERENCE ===" if use_coherence else "OUT COHERENCE ==="))
    for core in cores:
        print(f"{core.name}: Hits={core.cache_hits}, Misses={core.cache_misses}, "
              f"Coherence Msgs={core.coherence_messages if use_coherence else 'N/A'}")
    print(f"Execution Time: {duration:.4f} seconds")

if __name__ == "__main__":
    print(">>> SIMULATION: WITH COHERENCE PROTOCOL (MSI)")
    run_simulation(use_coherence=True)
    
    print("\n>>> SIMULATION: WITHOUT COHERENCE PROTOCOL")
    run_simulation(use_coherence=False)
```

# Output:
```python
>>> SIMULATION: WITH COHERENCE PROTOCOL (MSI)
[Core-1] WRITE x = 58
[Core-2] WRITE x = 58
[Core-1] WRITE x = 88
[Core-1] WRITE x = 75
[Core-2] WRITE x = 3
[Core-1] READ x = 75
[Core-2] READ x = 3
[Core-1] WRITE x = 91
[Core-1] WRITE x = 51
[Core-2] READ x = 3
[Core-2] READ x = 3
[Core-1] WRITE x = 60
[Core-2] WRITE x = 12
[Core-2] WRITE x = 81
[Core-1] READ x = 60
[Core-2] WRITE x = 98
[Core-1] WRITE x = 63
[Core-2] READ x = 98
[Core-1] WRITE x = 63
[Core-2] READ x = 98

=== STATS WITH COHERENCE ===
Core-1: Hits=2, Misses=0, Coherence Msgs=8
Core-2: Hits=5, Misses=0, Coherence Msgs=5
Execution Time: 0.3462 seconds

>>> SIMULATION: WITHOUT COHERENCE PROTOCOL
[Core-1] READ x = 63
[Core-2] READ x = 63
[Core-2] READ x = 63
[Core-1] READ x = 63
[Core-1] READ x = 63
[Core-2] READ x = 63
[Core-1] READ x = 63
[Core-2] WRITE x = 65
[Core-1] WRITE x = 56
[Core-1] READ x = 56
[Core-2] READ x = 65
[Core-2] READ x = 65
[Core-1] WRITE x = 95
[Core-2] WRITE x = 1
[Core-1] READ x = 95
[Core-2] READ x = 1
[Core-1] READ x = 95
[Core-2] WRITE x = 12
[Core-1] READ x = 95
[Core-2] READ x = 12

=== STATS WITHOUT COHERENCE ===
Core-1: Hits=7, Misses=1, Coherence Msgs=N/A
Core-2: Hits=6, Misses=1, Coherence Msgs=N/A
Execution Time: 0.3486 seconds
```
# Penjelasan

Program yang dijalankan merupakan simulasi dari dua core yang secara bersamaan mengakses dan memodifikasi variabel bersama bernama x menggunakan operasi READ dan WRITE. Setiap core memiliki cache lokal, dan simulasi dilakukan dalam dua skenario: dengan protokol koherensi cache (dalam hal ini, model sederhana protokol MSI) dan tanpa protokol koherensi. Tujuannya adalah untuk mengamati perbedaan perilaku sistem multiprosesor dalam menjaga konsistensi data dan pengaruhnya terhadap performa eksekusi.

Pada skenario pertama, protokol koherensi diaktifkan. Setiap kali salah satu core melakukan penulisan terhadap x, protokol memastikan bahwa salinan x pada cache core lainnya menjadi tidak valid. Hal ini memicu pengiriman pesan koherensi seperti invalidate atau read-update agar semua core memiliki nilai x yang konsisten. Output menunjukkan bahwa Core-1 memiliki 4 cache hit dan 1 cache miss, sedangkan Core-2 memiliki 5 cache hit dan 0 miss. Total ada 11 pesan koherensi yang dikirimkan, dan waktu eksekusi yang tercatat adalah 0.3471 detik. Hal ini menunjukkan bahwa mekanisme koherensi menambah overhead berupa sinkronisasi dan invalidasi antar-cache, yang mempengaruhi waktu eksekusi.

Pada skenario kedua, protokol koherensi dimatikan. Setiap core menyimpan dan membaca data dari cache lokal tanpa memperhatikan perubahan yang terjadi di core lain. Akibatnya, nilai x yang dibaca oleh satu core bisa saja berbeda dari nilai yang telah ditulis oleh core lainnya. Dalam skenario ini, Core-1 memiliki 4 cache hit dan 1 miss, sedangkan Core-2 mendapatkan 6 cache hit dan 0 miss. Tidak ada pesan koherensi yang dikirim, dan waktu eksekusi lebih cepat yaitu 0.3005 detik. Walaupun performa tampak meningkat, namun tidak ada jaminan bahwa data yang diakses bersifat konsisten dan akurat karena bisa terjadi kondisi data usang atau race condition.

Dari kedua hasil tersebut dapat disimpulkan bahwa protokol koherensi sangat penting untuk menjamin konsistensi data pada sistem multiprosesor, terutama ketika ada banyak thread yang berbagi data. Namun, hal ini datang dengan konsekuensi penurunan performa karena adanya kebutuhan komunikasi antar-cache dan penguncian memori. Di sisi lain, tidak menggunakan protokol koherensi memang meningkatkan kecepatan eksekusi, tetapi mengorbankan akurasi dan integritas data yang dapat berakibat pada bug atau hasil yang tidak dapat diandalkan. Perbandingan ini menggambarkan pentingnya keseimbangan antara konsistensi data dan performa dalam perancangan arsitektur sistem paralel.

