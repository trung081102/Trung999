# Trung999
validator in zetachain , Dym , STRK
#!/usr/bin/env python3
"""
Validator health checker (simple)
- Kiểm tra HTTP/JSON-RPC endpoints
- So sánh block height (nếu endpoint trả về)
- Gửi cảnh báo qua Telegram khi node không trả lời hoặc block height tụt
"""
import time
import requests
import logging
import sys
from typing import Dict, Optional

# CONFIG
CHECK_INTERVAL = 30  # seconds
TIMEOUT = 5  # requests timeout

# Endpoints: điền URL node của bạn ở đây
NODES = {
    "zetachain": "http://127.0.0.1:26657",   # ví dụ RPC
    "dym":       "http://127.0.0.1:26657",
    "strk":      "http://127.0.0.1:8545"     # có thể là HTTP JSON-RPC
}

# Telegram bot alert (optional)
TELEGRAM_BOT_TOKEN = ""   # "12345:ABCDEF..."
TELEGRAM_CHAT_ID = ""     # chat id (số hoặc @username)

LOGFILE = "validator_monitor.log"
logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(message)s",
                    handlers=[logging.FileHandler(LOGFILE), logging.StreamHandler(sys.stdout)])

def send_telegram(message: str):
    if not TELEGRAM_BOT_TOKEN or not TELEGRAM_CHAT_ID:
        logging.debug("Telegram not configured, skip sending")
        return
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    try:
        r = requests.post(url, json={"chat_id": TELEGRAM_CHAT_ID, "text": message}, timeout=5)
        if not r.ok:
            logging.warning("Telegram send failed: %s", r.text)
    except Exception as e:
        logging.exception("Error sending telegram")

def get_block_height_rpc(base_url: str) -> Optional[int]:
    """
    Try some common RPC calls to obtain block height.
    This will attempt Tendermint /status and Ethereum-style eth_blockNumber.
    """
    try:
        # Try Tendermint /status
        if base_url.endswith("/"):
            url = base_url + "status"
        else:
            url = base_url + "/status"
        r = requests.get(url, timeout=TIMEOUT)
        if r.ok:
            j = r.json()
            # tendermint path: result.sync_info.latest_block_height
            h = None
            if isinstance(j, dict):
                # try common shapes
                try:
                    h = int(j["result"]["sync_info"]["latest_block_height"])
                except Exception:
                    try:
                        h = int(j.get("result", {}).get("sync_info", {}).get("latest_block_height"))
                    except Exception:
                        h = None
            if h is not None:
                return h
    except Exception:
        pass

    # Try Ethereum JSON-RPC eth_blockNumber
    try:
        r = requests.post(base_url, json={"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}, timeout=TIMEOUT)
        if r.ok:
            j = r.json()
            if "result" in j and isinstance(j["result"], str):
                return int(j["result"], 16)
    except Exception:
        pass

    return None

def check_node(name: str, url: str, last_heights: Dict[str, Optional[int]]):
    ok = False
    try:
        h = get_block_height_rpc(url)
        if h is None:
            # try simple HTTP GET
            r = requests.get(url, timeout=TIMEOUT)
            ok = r.ok
            if ok:
                logging.info("[%s] endpoint OK (no block height)", name)
            else:
                logging.warning("[%s] endpoint returned status %s", name, r.status_code)
                send_telegram(f"[{name}] endpoint returned status {r.status_code}")
        else:
            ok = True
            logging.info("[%s] block height = %d", name, h)
            prev = last_heights.get(name)
            if prev is not None and h < prev:
                logging.error("[%s] block height decreased! prev=%s now=%d", name, prev, h)
                send_telegram(f"[{name}] block height decreased! prev={prev} now={h}")
            elif prev is not None and h == prev:
                logging.warning("[%s] block height unchanged (%d) — node may be stuck", name, h)
                send_telegram(f"[{name}] block height unchanged ({h}) — node may be stuck")
            last_heights[name] = h
    except Exception as e:
        logging.exception("[%s] check failed: %s", name, e)
        send_telegram(f"[{name}] check failed: {e}")
        ok = False
    return ok

def main():
    logging.info("Starting validator monitor")
    last_heights: Dict[str, Optional[int]] = {k: None for k in NODES.keys()}
    while True:
        for name, url in NODES.items():
            check_node(name, url, last_heights)
        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    main()
