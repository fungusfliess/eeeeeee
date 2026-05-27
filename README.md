# eeeeeee


```
\#!/usr/bin/env python3
"""
╔══════════════════════════════════════════════════════════════╗
║            B A T T L E S H I P  —  P 2 P                    ║
║                                                              ║
║  True peer-to-peer: no server process, no referee.          ║
║  Each peer owns its own board and validates incoming shots.  ║
║                                                              ║
║  Usage:                                                      ║
║    Host  :  python battleship.py --host [port]               ║
║    Join  :  python battleship.py --join <host_ip> [port]     ║
╚══════════════════════════════════════════════════════════════╝
"""

import socket
import json
import sys
import os
import time
import random
import threading
import hashlib
import secrets

# ─────────────────────────────────────────────────────────────
#  Constants
# ─────────────────────────────────────────────────────────────
_sock_buf: dict = {}   # socket fileno → leftover string
DEFAULT_PORT = 5555
SIZE  = 10
COLS  = "ABCDEFGHIJ"
SHIPS = [
    ("Carrier",    5),
    ("Battleship", 4),
    ("Cruiser",    3),
    ("Submarine",  3),
    ("Destroyer",  2),
]
EMPTY     = "."
SHIP_CELL = "S"
HIT_CELL  = "X"
MISS_CELL = "o"

# ─────────────────────────────────────────────────────────────
#  Terminal colours
# ─────────────────────────────────────────────────────────────
class C:
    RESET  = "\033[0m";  BOLD  = "\033[1m";  DIM   = "\033[2m"
    WATER  = "\033[38;5;33m";   SHIP  = "\033[38;5;243m"
    HIT    = "\033[38;5;196m";  MISS  = "\033[38;5;39m"
    SUNK   = "\033[38;5;208m";  TITLE = "\033[38;5;220m"
    INFO   = "\033[38;5;252m";  PROMPT= "\033[38;5;214m"
    OK     = "\033[38;5;82m";   ERR   = "\033[38;5;196m"
    COORD  = "\033[38;5;159m";  PEER  = "\033[38;5;197m"

def clear():  os.system("cls" if os.name == "nt" else "clear")
def sleep(s=0.4): time.sleep(s)

# ─────────────────────────────────────────────────────────────
#  Network helpers  (newline-delimited JSON, line-buffered)
# ─────────────────────────────────────────────────────────────
def net_send(sock, obj):
    sock.sendall((json.dumps(obj, separators=(',', ':')) + "\n").encode())

def net_recv(sock):
    """Read exactly one JSON message, buffering any leftover bytes."""
    fd = sock.fileno()
    if fd not in _sock_buf:
        _sock_buf[fd] = ""
    while True:
        idx = _sock_buf[fd].find("\n")
        if idx != -1:
            line = _sock_buf[fd][:idx]
            _sock_buf[fd] = _sock_buf[fd][idx + 1:]
            return json.loads(line)
        chunk = sock.recv(4096)
        if not chunk:
            raise ConnectionResetError("Peer disconnected")
        _sock_buf[fd] += chunk.decode()

# ─────────────────────────────────────────────────────────────
#  Board helpers
# ─────────────────────────────────────────────────────────────
def make_board():
    return [[EMPTY] * SIZE for _ in range(SIZE)]

def render_cell(cell, hide=False):
    if cell == SHIP_CELL: return (C.WATER+"~") if hide else (C.SHIP+"■")
    if cell == HIT_CELL:  return C.HIT  + "✕"
    if cell == MISS_CELL: return C.MISS + "·"
    return C.WATER + "~"

def print_boards(mine, enemy, my_label="YOU", en_label="OPPONENT"):
    w = 24
    print()
    print(f"  {C.BOLD}{C.OK}{my_label:^{w}}{C.RESET}       "
          f"  {C.BOLD}{C.PEER}{en_label:^{w}}{C.RESET}")
    hdr = "  ".join(C.COORD + ch + C.RESET for ch in COLS)
    pad = "    "
    print(f"  {pad}{hdr}        {pad}{hdr}")
    for r in range(SIZE):
        rn = C.COORD + f"{r+1:>2}" + C.RESET
        mr = "  ".join(render_cell(mine[r][c])        + C.RESET for c in range(SIZE))
        er = "  ".join(render_cell(enemy[r][c], True) + C.RESET for c in range(SIZE))
        print(f"  {rn}  {mr}        {rn}  {er}")
    print()

def print_own_board(board, label="YOUR FLEET"):
    hdr = "  ".join(C.COORD + ch + C.RESET for ch in COLS)
    print(f"\n  {C.BOLD}{C.TITLE}{label}{C.RESET}")
    print(f"      {hdr}")
    for r in range(SIZE):
        rn = C.COORD + f"{r+1:>2}" + C.RESET
        row = "  ".join(render_cell(board[r][c]) + C.RESET for c in range(SIZE))
        print(f"  {rn}  {row}")
    print()

def can_place(board, row, col, length, h):
    coords = []
    for i in range(length):
        r = row + (0 if h else i)
        c = col + (i if h else 0)
        if not (0 <= r < SIZE and 0 <= c < SIZE): return None
        if board[r][c] != EMPTY: return None
        coords.append((r, c))
    return coords

def place_ship(board, row, col, length, h):
    coords = can_place(board, row, col, length, h)
    if coords is None: return False
    for r, c in coords: board[r][c] = SHIP_CELL
    return True

def random_place(board):
    for _, length in SHIPS:
        ok = False
        while not ok:
            h = random.choice([True, False])
            r = random.randint(0, SIZE-1)
            c = random.randint(0, SIZE-1)
            ok = place_ship(board, r, c, length, h)

def build_tracker(board):
    vis = [[False]*SIZE for _ in range(SIZE)]
    ships = []
    for r in range(SIZE):
        for c in range(SIZE):
            if board[r][c] == SHIP_CELL and not vis[r][c]:
                cells, q = [], [(r,c)]
                while q:
                    cr,cc = q.pop()
                    if vis[cr][cc]: continue
                    vis[cr][cc] = True
                    cells.append((cr,cc))
                    for dr,dc in [(0,1),(0,-1),(1,0),(-1,0)]:
                        nr,nc = cr+dr,cc+dc
                        if 0<=nr<SIZE and 0<=nc<SIZE and not vis[nr][nc] and board[nr][nc]==SHIP_CELL:
                            q.append((nr,nc))
                ships.append(cells)
    return ships

def check_sunk(cells, board):
    return all(board[r][c] == HIT_CELL for r,c in cells)

def all_sunk(tracker, board):
    return all(check_sunk(s, board) for s in tracker)

def ship_name(size):
    return {5:"Carrier", 4:"Battleship", 3:"Cruiser/Sub", 2:"Destroyer"}.get(size, "Ship")

def print_fleet_status(my_tracker, my_board, enemy_ship_sunk_list):
    """Print a two-column fleet status table: your ships left | enemy ships sunk."""
    my_sorted = sorted(my_tracker, key=lambda s: -len(s))  # largest first = matches SHIPS order
    print(f"  {C.OK}{C.BOLD}YOUR FLEET{C.RESET:<20}          "
          f"{C.PEER}{C.BOLD}ENEMY FLEET{C.RESET}")
    for i, (name, _) in enumerate(SHIPS):
        # my ship status
        if i < len(my_sorted):
            sunk = check_sunk(my_sorted[i], my_board)
            if sunk:
                my_part = f"  {C.HIT}✕ {name:<12}{C.RESET}{C.DIM} SUNK{C.RESET}"
            else:
                my_part = f"  {C.OK}■ {name:<12}{C.RESET}{C.DIM} afloat{C.RESET}"
        else:
            my_part = ""
        # enemy ship status
        if enemy_ship_sunk_list[i]:
            en_part = f"  {C.HIT}✕ {name:<12}{C.RESET}{C.DIM} SUNK{C.RESET}"
        else:
            en_part = f"  {C.PEER}? {name:<12}{C.RESET}{C.DIM} unknown{C.RESET}"
        print(f"{my_part:<52}{en_part}")
    print()

def parse_coord(raw):
    raw = raw.strip().upper()
    if not raw or raw[0] not in COLS: return None, None
    col = COLS.index(raw[0])
    try:   row = int(raw[1:]) - 1
    except ValueError: return None, None
    if not (0 <= row < SIZE): return None, None
    return row, col

# ─────────────────────────────────────────────────────────────
#  Commitment scheme  (prevents cheating on board reveal)
#  Each peer hashes their board + a random nonce, exchanges
#  the hash before placement, then reveals board+nonce after
#  game over so the opponent can verify no cheating occurred.
# ─────────────────────────────────────────────────────────────
def make_commitment(board, nonce):
    data = json.dumps(board, separators=(',',':')) + nonce
    return hashlib.sha256(data.encode()).hexdigest()

def verify_commitment(board, nonce, commitment):
    return make_commitment(board, nonce) == commitment

# ─────────────────────────────────────────────────────────────
#  Title
# ─────────────────────────────────────────────────────────────
def title_screen(role):
    clear()
    color = C.OK if role == "host" else C.PEER
    role_label = "HOST (Player 1)" if role == "host" else "GUEST (Player 2)"
    print(f"""
{C.TITLE}{C.BOLD}
  ██████╗  █████╗ ████████╗████████╗██╗     ███████╗███████╗██╗  ██╗██╗██████╗ 
  ██╔══██╗██╔══██╗╚══██╔══╝╚══██╔══╝██║     ██╔════╝██╔════╝██║  ██║██║██╔══██╗
  ██████╔╝███████║   ██║      ██║   ██║     █████╗  ███████╗███████║██║██████╔╝
  ██╔══██╗██╔══██║   ██║      ██║   ██║     ██╔══╝  ╚════██║██╔══██║██║██╔═══╝ 
  ██████╔╝██║  ██║   ██║      ██║   ███████╗███████╗███████║██║  ██║██║██║     
  ╚═════╝ ╚═╝  ╚═╝   ╚═╝      ╚═╝   ╚══════╝╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚═╝{C.RESET}
""")
    print(f"  {C.DIM}{'─'*68}{C.RESET}")
    print(f"  {color}{C.BOLD}  {role_label}{C.RESET}")
    print(f"  {C.DIM}  Peer-to-Peer — no server needed{C.RESET}")
    print(f"  {C.DIM}{'─'*68}{C.RESET}\n")

# ─────────────────────────────────────────────────────────────
#  Ship placement UI
# ─────────────────────────────────────────────────────────────
def do_placement():
    board = make_board()
    print(f"\n{C.TITLE}{C.BOLD}  ═══ PLACE YOUR SHIPS ═══{C.RESET}\n")
    print(f"  {C.INFO}Format: {C.COORD}C4 H{C.INFO} (column, row, H or V).  "
          f"Type {C.PROMPT}AUTO{C.INFO} to place randomly.{C.RESET}\n")

    for name, length in SHIPS:
        while True:
            print_own_board(board, "YOUR FLEET")
            print(f"  {C.BOLD}{C.TITLE}Place {name}{C.RESET}  {C.DIM}(size {length}){C.RESET}")
            raw = input(f"  {C.PROMPT}▶ {C.RESET}").strip().upper()

            if raw == "AUTO":
                random_place(board)
                print(f"\n  {C.OK}Ships placed randomly!{C.RESET}\n")
                sleep(0.5)
                return board

            parts = raw.split()
            if len(parts) != 2:
                print(f"  {C.ERR}Try: C4 H{C.RESET}\n"); continue
            coord, orient = parts
            if orient not in ("H","V"):
                print(f"  {C.ERR}Orientation must be H or V{C.RESET}\n"); continue
            if len(coord) < 2 or coord[0] not in COLS:
                print(f"  {C.ERR}Invalid coordinate{C.RESET}\n"); continue
            col = COLS.index(coord[0])
            try:   row = int(coord[1:]) - 1
            except ValueError:
                print(f"  {C.ERR}Invalid row number{C.RESET}\n"); continue

            if place_ship(board, row, col, length, orient == "H"):
                print(f"  {C.OK}✓ {name} placed!{C.RESET}\n")
                sleep(0.2)
                break
            else:
                print(f"  {C.ERR}Can't place there — out of bounds or overlap.{C.RESET}\n")
    return board

# ─────────────────────────────────────────────────────────────
#  Coin-flip: both peers contribute randomness fairly
#  Host picks a random bit and commits; guest sends their bit;
#  host reveals → XOR decides who goes first (0=host, 1=guest)
# ─────────────────────────────────────────────────────────────
def coinflip_host(sock):
    """Host side: commit → receive guest bit → reveal → return winner (0=host,1=guest)"""
    my_bit   = random.randint(0, 1)
    my_nonce = secrets.token_hex(16)
    commit   = hashlib.sha256(f"{my_bit}{my_nonce}".encode()).hexdigest()
    net_send(sock, {"type": "coin_commit", "commit": commit})
    msg = net_recv(sock)
    guest_bit = msg["bit"]
    net_send(sock, {"type": "coin_reveal", "bit": my_bit, "nonce": my_nonce})
    first = my_bit ^ guest_bit   # 0 → host first, 1 → guest first
    return first

def coinflip_guest(sock):
    """Guest side: receive commit → send bit → receive reveal → verify → return winner"""
    msg    = net_recv(sock)       # coin_commit
    commit = msg["commit"]
    my_bit = random.randint(0, 1)
    net_send(sock, {"type": "coin_bit", "bit": my_bit})
    reveal = net_recv(sock)       # coin_reveal
    host_bit   = reveal["bit"]
    host_nonce = reveal["nonce"]
    expected = hashlib.sha256(f"{host_bit}{host_nonce}".encode()).hexdigest()
    if expected != commit:
        raise ValueError("Host cheated on the coin flip!")
    first = host_bit ^ my_bit
    return first   # 0=host first, 1=guest first

# ─────────────────────────────────────────────────────────────
#  Main game loop  (shared by both peers)
# ─────────────────────────────────────────────────────────────
def play(sock, my_board, my_role, first):
    """
    my_role: "host" (player 0) or "guest" (player 1)
    first:    0 → host attacks first,  1 → guest attacks first
    """
    my_id    = 0 if my_role == "host" else 1
    their_id = 1 - my_id
    # I attack first if first == my_id
    my_turn  = (first == my_id)

    my_tracker    = build_tracker(my_board)
    enemy_board   = make_board()        # tracking board for enemy (their ships hidden)
    round_num     = 0
    my_hits       = 0
    their_hits    = 0

    # ── message pump for incoming shots while we wait ─────────────────────────
    # We run a recv-thread so that when it's our turn we still get the
    # enemy shot cleanly after we send ours (prevents deadlock).
    incoming = []
    incoming_event = threading.Event()
    recv_lock = threading.Lock()
    stop_recv = threading.Event()

    def recv_thread():
        while not stop_recv.is_set():
            try:
                sock.settimeout(1.0)
                msg = net_recv(sock)
                with recv_lock:
                    incoming.append(msg)
                incoming_event.set()
            except socket.timeout:
                continue
            except (ConnectionResetError, OSError):
                break
        sock.settimeout(None)

    rt = threading.Thread(target=recv_thread, daemon=True)
    rt.start()

    def wait_msg():
        """Block until a message arrives from the recv thread."""
        while True:
            incoming_event.wait()
            with recv_lock:
                if incoming:
                    msg = incoming.pop(0)
                    if not incoming:
                        incoming_event.clear()
                    return msg
            incoming_event.clear()

    # ── game loop ──────────────────────────────────────────────────────────────
    while True:
        round_num += 1
        clear()
        print(f"\n  {C.TITLE}{C.BOLD}═══ ROUND {round_num} ═══{C.RESET}\n")
        ship_status_bar(my_tracker,  my_board,    "YOUR FLEET  ", C.OK)
        print()
        print_boards(my_board, enemy_board, my_label="YOU", en_label="OPPONENT")

        if my_turn:
            # ── I ATTACK ──────────────────────────────────────────────────────
            print(f"  {C.BOLD}{C.PROMPT}YOUR TURN — Fire!{C.RESET}")
            while True:
                raw = input(f"  {C.PROMPT}▶ Target (e.g. E6) : {C.RESET}").strip()
                if raw.lower() in ("q","quit","exit"):
                    net_send(sock, {"type":"quit"})
                    stop_recv.set()
                    print(f"\n  {C.DIM}You surrendered. Goodbye.{C.RESET}\n")
                    return
                r, c = parse_coord(raw)
                if r is None:
                    print(f"  {C.ERR}Invalid coordinate.{C.RESET}"); continue
                if enemy_board[r][c] in (HIT_CELL, MISS_CELL):
                    print(f"  {C.ERR}Already fired there!{C.RESET}"); continue
                break

            net_send(sock, {"type":"fire","row":r,"col":c})

            # Wait for result from peer (they evaluate the shot against their board)
            result_msg = wait_msg()
            if result_msg["type"] == "quit":
                stop_recv.set()
                print(f"\n  {C.SUNK}{C.BOLD}  ★ Opponent surrendered! You win! ★{C.RESET}\n")
                return
            if result_msg["type"] == "game_over":
                stop_recv.set()
                _end_screen(result_msg, my_id, my_board, enemy_board, round_num)
                return

            # Apply result to our tracking board
            result = result_msg["result"]
            if result in ("hit","sunk"):
                enemy_board[r][c] = HIT_CELL
                my_hits += 1
                if result == "sunk":
                    print(f"\n  {C.SUNK}{C.BOLD}  ★ You sunk their {result_msg['sunk_name']}! ★{C.RESET}")
                else:
                    print(f"\n  {C.HIT}{C.BOLD}  ✕ HIT at {COLS[c]}{r+1}!{C.RESET}")
            else:
                enemy_board[r][c] = MISS_CELL
                print(f"\n  {C.MISS}  · Miss at {COLS[c]}{r+1}.{C.RESET}")

            sleep(0.8)
            my_turn = False

        else:
            # ── THEY ATTACK ───────────────────────────────────────────────────
            print(f"  {C.DIM}  Waiting for opponent's shot...{C.RESET}\n")
            fire_msg = wait_msg()

            if fire_msg["type"] == "quit":
                stop_recv.set()
                print(f"\n  {C.SUNK}{C.BOLD}  ★ Opponent surrendered! You win! ★{C.RESET}\n")
                return

            tr = fire_msg["row"]
            tc = fire_msg["col"]
            coord_s = f"{COLS[tc]}{tr+1}"

            # Evaluate shot against OUR board
            if my_board[tr][tc] == SHIP_CELL:
                my_board[tr][tc] = HIT_CELL
                their_hits += 1
                # check sunk
                sunk_name = None
                for s in my_tracker:
                    if (tr,tc) in s and check_sunk(s, my_board):
                        sunk_name = ship_name(len(s))
                        break
                result = "sunk" if sunk_name else "hit"

                if sunk_name:
                    print(f"\n  {C.ERR}{C.BOLD}  ★ Opponent sunk your {sunk_name}! ★{C.RESET}")
                else:
                    print(f"\n  {C.ERR}  Enemy hit your ship at {coord_s}!{C.RESET}")

                # Check if we lost
                if all_sunk(my_tracker, my_board):
                    result_payload = {
                        "type":      "fire_result",
                        "result":    result,
                        "sunk_name": sunk_name,
                    }
                    net_send(sock, result_payload)
                    # Send game_over
                    over = {
                        "type":    "game_over",
                        "winner":  their_id,
                        "round":   round_num,
                        "my_hits": their_hits,
                    }
                    net_send(sock, over)
                    stop_recv.set()
                    _end_screen(over, my_id, my_board, enemy_board, round_num)
                    return

                net_send(sock, {"type":"fire_result","result":result,"sunk_name":sunk_name})

            else:
                my_board[tr][tc] = MISS_CELL
                print(f"\n  {C.MISS}  Enemy missed at {coord_s}.{C.RESET}")
                net_send(sock, {"type":"fire_result","result":"miss","sunk_name":None})

            sleep(0.8)
            my_turn = True

    stop_recv.set()


def _end_screen(msg, my_id, my_board, enemy_board, round_num):
    winner = msg.get("winner", -1)
    clear()
    print_boards(my_board, enemy_board)
    if winner == my_id:
        print(f"\n  {C.OK}{C.BOLD}{'═'*52}")
        print(f"   🏆  VICTORY! You sank the entire enemy fleet!")
    else:
        print(f"\n  {C.ERR}{C.BOLD}{'═'*52}")
        print(f"   💀  DEFEAT. Your fleet has been destroyed.")
    print(f"   Rounds played: {round_num}")
    print(f"{'═'*52}{C.RESET}\n")

# ─────────────────────────────────────────────────────────────
#  Handshake: exchange board commitments, then sync ready
# ─────────────────────────────────────────────────────────────
def handshake_host(sock, board, nonce, commit):
    """Send our commitment, receive theirs, both send 'ready'."""
    net_send(sock, {"type":"commit","commit":commit})
    their_commit_msg = net_recv(sock)
    their_commit = their_commit_msg["commit"]
    # both signal ready
    net_send(sock, {"type":"ready"})
    net_recv(sock)   # wait for their ready
    return their_commit

def handshake_guest(sock, board, nonce, commit):
    their_commit_msg = net_recv(sock)
    their_commit = their_commit_msg["commit"]
    net_send(sock, {"type":"commit","commit":commit})
    net_recv(sock)   # wait for host ready
    net_send(sock, {"type":"ready"})
    return their_commit

# ─────────────────────────────────────────────────────────────
#  Entry points
# ─────────────────────────────────────────────────────────────
def run_host(port):
    title_screen("host")
    print(f"  {C.INFO}Listening on port {C.COORD}{port}{C.RESET} — waiting for opponent...\n")

    srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(("0.0.0.0", port))
    srv.listen(1)

    # Place ships while waiting (in background? No — place first, then wait)
    # Actually: place ships, THEN listen so the opponent can also be placing
    my_board = do_placement()
    my_nonce = secrets.token_hex(16)
    my_commit = make_commitment(my_board, my_nonce)

    print(f"\n  {C.DIM}Waiting for opponent to connect...{C.RESET}")
    conn, addr = srv.accept()
    srv.close()
    print(f"  {C.OK}Opponent connected from {addr[0]}!{C.RESET}\n")
    sleep(0.5)

    their_commit = handshake_host(conn, my_board, my_nonce, my_commit)

    # Coin flip — host coordinates
    first = coinflip_host(conn)
    first_label = "You go first!" if first == 0 else "Opponent goes first!"
    print(f"  {C.TITLE}{C.BOLD}  {first_label}{C.RESET}\n")
    sleep(1.2)

    play(conn, my_board, "host", first)

    # Post-game: reveal boards for anti-cheat verification
    print(f"\n  {C.DIM}Exchanging boards for verification...{C.RESET}")
    net_send(conn, {"type":"reveal","board":my_board,"nonce":my_nonce})
    try:
        their_reveal = net_recv(conn)
        ok = verify_commitment(their_reveal["board"], their_reveal["nonce"], their_commit)
        if ok:
            print(f"  {C.OK}✓ Opponent's board verified — no cheating detected.{C.RESET}\n")
        else:
            print(f"  {C.ERR}⚠ Board verification FAILED — opponent may have cheated!{C.RESET}\n")
    except Exception:
        pass
    conn.close()

def run_guest(host_ip, port):
    title_screen("guest")
    print(f"  {C.INFO}Connecting to {C.COORD}{host_ip}:{port}{C.RESET}...\n")

    my_board = do_placement()
    my_nonce = secrets.token_hex(16)
    my_commit = make_commitment(my_board, my_nonce)

    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((host_ip, port))
    except ConnectionRefusedError:
        print(f"  {C.ERR}Could not connect to {host_ip}:{port}{C.RESET}")
        print(f"  {C.DIM}Make sure the host is running and the port is correct.{C.RESET}\n")
        sys.exit(1)

    print(f"  {C.OK}Connected!{C.RESET}\n")
    sleep(0.5)

    their_commit = handshake_guest(sock, my_board, my_nonce, my_commit)

    first = coinflip_guest(sock)
    first_label = "Opponent goes first!" if first == 0 else "You go first!"
    print(f"  {C.TITLE}{C.BOLD}  {first_label}{C.RESET}\n")
    sleep(1.2)

    play(sock, my_board, "guest", first)

    # Post-game reveal
    print(f"\n  {C.DIM}Exchanging boards for verification...{C.RESET}")
    net_send(sock, {"type":"reveal","board":my_board,"nonce":my_nonce})
    try:
        their_reveal = net_recv(sock)
        ok = verify_commitment(their_reveal["board"], their_reveal["nonce"], their_commit)
        if ok:
            print(f"  {C.OK}✓ Opponent's board verified — no cheating detected.{C.RESET}\n")
        else:
            print(f"  {C.ERR}⚠ Board verification FAILED — opponent may have cheated!{C.RESET}\n")
    except Exception:
        pass
    sock.close()

# ─────────────────────────────────────────────────────────────
#  CLI
# ─────────────────────────────────────────────────────────────
def usage():
    print(f"""
{C.TITLE}{C.BOLD}Battleship P2P{C.RESET}

  {C.PROMPT}Host a game:{C.RESET}
    python battleship.py --host [port]          (default port {DEFAULT_PORT})

  {C.PROMPT}Join a game:{C.RESET}
    python battleship.py --join <ip> [port]

  {C.DIM}Examples:
    python battleship.py --host
    python battleship.py --host 7777
    python battleship.py --join 192.168.1.42
    python battleship.py --join 192.168.1.42 7777{C.RESET}
""")

if __name__ == "__main__":
    args = sys.argv[1:]
    try:
        if not args or args[0] in ("-h","--help","help"):
            usage()
        elif args[0] == "--host":
            port = int(args[1]) if len(args) > 1 else DEFAULT_PORT
            run_host(port)
        elif args[0] == "--join":
            if len(args) < 2:
                print(f"  {C.ERR}Usage: python battleship.py --join <host_ip> [port]{C.RESET}\n")
                sys.exit(1)
            host_ip = args[1]
            port    = int(args[2]) if len(args) > 2 else DEFAULT_PORT
            run_guest(host_ip, port)
        else:
            usage()
    except KeyboardInterrupt:
        print(f"\n\n  {C.DIM}Interrupted. Goodbye!{C.RESET}\n")
    except ConnectionResetError:
        print(f"\n  {C.ERR}Peer disconnected unexpectedly.{C.RESET}\n")
    except ValueError as e:
        print(f"\n  {C.ERR}Error: {e}{C.RESET}\n")
```
