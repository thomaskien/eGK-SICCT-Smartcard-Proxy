# installation unter windows

- am einfachsten .exe runterladen

Code selbst überprüfen:
- python installieren von python.org
- powershell öffnen
- untiges pasten und sich so selbst eine .exe bauen am einfachsten

<pre>
$ErrorActionPreference = "Stop"

$proj = "$HOME\Desktop\sicct_server_windows"
New-Item -ItemType Directory -Force -Path $proj | Out-Null
Set-Location $proj

py -3 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install pyscard pyinstaller

@'
#!/usr/bin/env python3
import socket
import threading
import itertools
from dataclasses import dataclass, field
from typing import Optional

from smartcard.System import readers
from smartcard.CardConnection import CardConnection

# Für nur lokalen Zugriff auf demselben PC:
# HOST = "127.0.0.1"
HOST = "0.0.0.0"
PORT = 4742

# Windows: zuerst eher locker starten.
# Wenn nur ein Reader vorhanden ist, ist das normalerweise Index 0.
READER_INDEX = 0
EXPECTED_READER_SUBSTR = ""

# Verzögerung für das Slot-Removed-Event nach EJECT
SLOT_REMOVED_EVENT_DELAY_SEC = 0.05

# Reader-/Kartenzustand (global, da physisch ein Gerät)
reader_lock = threading.RLock()
current_conn = None
current_atr = None

last_display_text = ""
idle_display_text = ""


@dataclass
class ConnState:
    conn: socket.socket
    addr: tuple
    conn_id: int
    lock: threading.RLock = field(default_factory=threading.RLock)
    session_sid: Optional[str] = None
    username: str = "test"
    closed: bool = False
    event_seq: int = 1


session_counter = itertools.count(1)


def hexdump(data: bytes) -> str:
    lines = []
    for i in range(0, len(data), 16):
        chunk = data[i:i + 16]
        hexs = " ".join(f"{b:02X}" for b in chunk)
        txt = "".join(chr(b) if 32 <= b < 127 else "." for b in chunk)
        lines.append(f"{i:04X}  {hexs:<47}  {txt}")
    return "\n".join(lines)


def list_readers_safe():
    try:
        return list(readers())
    except Exception as e:
        print("Reader-Liste Fehler:", e)
        return []


def find_reader():
    rs = list_readers_safe()
    if not rs:
        return None

    if EXPECTED_READER_SUBSTR:
        for r in rs:
            if EXPECTED_READER_SUBSTR.lower() in str(r).lower():
                return r

    if 0 <= READER_INDEX < len(rs):
        return rs[READER_INDEX]

    return None


def connect_card_with_fallback(conn):
    """
    Windows-robuster:
    erst T=1, dann T=0, dann automatische Aushandlung.
    """
    last_err = None

    for proto in (CardConnection.T1_protocol, CardConnection.T0_protocol, None):
        try:
            if proto is None:
                conn.connect()
            else:
                conn.connect(proto)
            return
        except Exception as e:
            last_err = e

    raise last_err if last_err is not None else RuntimeError("Unbekannter Connect-Fehler")


def disconnect_current():
    global current_conn, current_atr
    if current_conn is not None:
        try:
            current_conn.disconnect()
        except Exception:
            pass
    current_conn = None
    current_atr = None


def cleanup_reader_state():
    with reader_lock:
        disconnect_current()


def build_ctsess_do(user: str, sid: str) -> bytes:
    ub = user.encode("ascii", errors="ignore")
    sb = sid.encode("ascii", errors="ignore")
    value = (
        bytes([0x13, len(ub)]) + ub +
        b"\x13\x00" +
        bytes([0x13, len(sb)]) + sb
    )
    return b"\x69" + bytes([len(value)]) + value


def parse_ctsess_do(data: bytes):
    user = ""
    password = ""
    sid = ""

    try:
        if not data or data[0] != 0x69 or len(data) < 2:
            return user, password, sid

        total_len = data[1]
        payload = data[2:2 + total_len]

        values = []
        i = 0
        while i + 1 < len(payload):
            tag = payload[i]
            i += 1
            if tag != 0x13:
                break
            length = payload[i]
            i += 1
            value = payload[i:i + length]
            i += length
            values.append(value.decode("ascii", errors="ignore"))

        if len(values) >= 1:
            user = values[0]
        if len(values) >= 2:
            password = values[1]
        if len(values) >= 3:
            sid = values[2]
    except Exception:
        pass

    return user, password, sid


def card_status_byte() -> int:
    """
    SICCT ICC Status Byte:
      0x00 = keine Karte
      0x15 = Karte vorhanden, powered, specific mode
    """
    global current_conn, current_atr

    with reader_lock:
        if current_conn is not None and current_atr:
            return 0x15

        r = find_reader()
        if r is None:
            return 0x00

        conn = r.createConnection()
        try:
            connect_card_with_fallback(conn)
            atr = bytes(conn.getATR())
            try:
                conn.disconnect()
            except Exception:
                pass
            return 0x15 if atr else 0x00
        except Exception:
            try:
                conn.disconnect()
            except Exception:
                pass
            return 0x00


def request_icc() -> bool:
    global current_conn, current_atr

    with reader_lock:
        disconnect_current()
        r = find_reader()
        if r is None:
            print("request_icc: kein passender Reader gefunden")
            return False

        conn = r.createConnection()
        try:
            connect_card_with_fallback(conn)
            atr = bytes(conn.getATR())
            current_conn = conn
            current_atr = atr
            return True
        except Exception as e:
            print("request_icc error:", e)
            try:
                conn.disconnect()
            except Exception:
                pass
            current_conn = None
            current_atr = None
            return False


def reset_icc(warm: bool) -> tuple[bool, bytes]:
    global current_conn, current_atr

    with reader_lock:
        disconnect_current()
        r = find_reader()
        if r is None:
            print("reset_icc: kein passender Reader gefunden")
            return False, b""

        conn = r.createConnection()
        try:
            connect_card_with_fallback(conn)
            atr = bytes(conn.getATR())
            current_conn = conn
            current_atr = atr
            return True, atr
        except Exception as e:
            print("reset_icc error:", e)
            try:
                conn.disconnect()
            except Exception:
                pass
            current_conn = None
            current_atr = None
            return False, b""


def build_response(req: bytes, rapdu: bytes) -> bytes:
    if len(req) < 10:
        raise ValueError("Request zu kurz für SICCT-Envelope")
    if len(rapdu) > 0xFFFF:
        raise ValueError(f"RAPDU zu lang: {len(rapdu)} Bytes")

    env = bytearray(req[:10])
    env[0] = 0x6B
    env[8] = (len(rapdu) >> 8) & 0xFF
    env[9] = len(rapdu) & 0xFF
    return bytes(env) + rapdu


def build_event_envelope(state: ConnState, payload: bytes) -> bytes:
    seq = state.event_seq & 0xFFFF
    state.event_seq += 1

    env = bytearray(10)
    env[0] = 0x6B
    env[1] = 0x00
    env[2] = 0x00
    env[3] = (seq >> 8) & 0xFF
    env[4] = seq & 0xFF
    env[5] = 0x00
    env[6] = 0x00
    env[7] = 0x00
    env[8] = (len(payload) >> 8) & 0xFF
    env[9] = len(payload) & 0xFF
    return bytes(env) + payload


def tlv(tag: bytes, value: bytes) -> bytes:
    l = len(value)
    if l <= 0x7F:
        return tag + bytes([l]) + value
    if l <= 0xFF:
        return tag + b"\x81" + bytes([l]) + value
    if l <= 0xFFFF:
        return tag + b"\x82" + bytes([(l >> 8) & 0xFF, l & 0xFF]) + value
    raise ValueError("TLV too long")


def parse_simple_tlvs(data: bytes):
    i = 0
    out = []
    n = len(data)

    while i < n:
        tag_start = data[i]
        i += 1
        tag = bytes([tag_start])

        if (tag_start & 0x1F) == 0x1F:
            while i < n:
                b = data[i]
                tag += bytes([b])
                i += 1
                if (b & 0x80) == 0:
                    break

        if i >= n:
            break

        l1 = data[i]
        i += 1

        if l1 <= 0x7F:
            length = l1
        elif l1 == 0x81:
            if i >= n:
                break
            length = data[i]
            i += 1
        elif l1 == 0x82:
            if i + 1 >= n:
                break
            length = (data[i] << 8) | data[i + 1]
            i += 2
        else:
            break

        if i + length > n:
            break

        value = data[i:i + length]
        i += length
        out.append((tag, value))

    return out


def decode_best_effort_text(value: bytes) -> str:
    for enc in ("utf-8", "latin-1", "cp1252"):
        try:
            return value.decode(enc)
        except Exception:
            pass
    return repr(value)


def close_socket_hard(state: ConnState):
    with state.lock:
        if state.closed:
            return
        state.closed = True
        try:
            state.conn.shutdown(socket.SHUT_RDWR)
        except Exception:
            pass
        try:
            state.conn.close()
        except Exception:
            pass


def send_slot_removed_event(state: ConnState, fu_number: int):
    with state.lock:
        if state.closed or state.session_sid is None:
            return

        payload = bytes([
            0x85, 0x02,
            (fu_number >> 8) & 0xFF,
            fu_number & 0xFF
        ])
        pkt = build_event_envelope(state, payload)

        print(f"Sende Slotereignis Karte entfernt (85) an {state.addr}, FU={fu_number}")
        print(hexdump(pkt))

        try:
            state.conn.sendall(pkt)
        except Exception as e:
            print("Slot-Removed-Event senden fehlgeschlagen:", e)


def schedule_slot_removed_event(state: ConnState, fu_number: int):
    t = threading.Timer(
        SLOT_REMOVED_EVENT_DELAY_SEC,
        send_slot_removed_event,
        args=(state, fu_number)
    )
    t.daemon = True
    t.start()


def rapdu_init_session(apdu: bytes, state: ConnState) -> bytes:
    cleanup_reader_state()

    if len(apdu) < 5:
        return b"\x67\x00"

    lc = apdu[4]
    if 5 + lc > len(apdu):
        return b"\x67\x00"

    data = apdu[5:5 + lc]
    user, password, sid_req = parse_ctsess_do(data)

    if not user:
        user = "test"

    with state.lock:
        if state.session_sid is not None:
            return b"\x69\x00"

        sid = str(next(session_counter))
        state.session_sid = sid
        state.username = user
        state.event_seq = 1

    print(f"INIT CT SESSION: user={user!r}, neue session={sid!r}, alt_req_sid={sid_req!r}")
    return build_ctsess_do(user, sid) + b"\x90\x00"


def rapdu_close_session(apdu: bytes, state: ConnState) -> bytes:
    lc = apdu[4] if len(apdu) >= 5 else 0
    data = apdu[5:5 + lc] if len(apdu) >= 5 and 5 + lc <= len(apdu) else b""
    user, password, sid = parse_ctsess_do(data)

    with state.lock:
        if state.session_sid is None:
            return b"\x69\x00"
        if sid and sid != state.session_sid:
            print(f"CLOSE CT SESSION: falsche sid={sid!r}, erwartet={state.session_sid!r}")
            return b"\x6F\x00"

        print(f"CLOSE CT SESSION: sid={state.session_sid!r}")
        state.session_sid = None

    cleanup_reader_state()
    return b"\x90\x00"


def rapdu_request_icc(apdu: bytes, state: ConnState) -> bytes:
    with state.lock:
        if state.session_sid is None:
            return b"\x69\x00"

    if request_icc():
        return b"\x90\x01"
    return b"\x64\xA1"


def rapdu_reset_ct_icc(apdu: bytes, state: ConnState) -> bytes:
    with state.lock:
        if state.session_sid is None:
            return b"\x69\x00"

    if len(apdu) < 4:
        return b"\x67\x00"

    p1 = apdu[2]
    p2 = apdu[3]

    if p1 != 0x01:
        return b"\x6A\x00"

    warm = bool(p2 & 0x80)
    want_status_do = bool(p2 & 0x10)
    want_reset_info = p2 & 0x0F

    ok, atr = reset_icc(warm=warm)
    if not ok:
        return b"\x64\x00"

    body = b""

    if want_status_do:
        body += b"\x80\x01\x15"

    if want_reset_info == 0x01:
        body += tlv(b"\x5F\x41", atr)
    elif want_reset_info == 0x02:
        hb = b""
        try:
            if len(atr) >= 2:
                k = atr[1] & 0x0F
                hb = atr[-(k + 1):-1] if k > 0 else b""
        except Exception:
            hb = b""
        body += tlv(b"\x5F\x52", hb)
    elif want_reset_info == 0x04:
        return b"\x6A\x80"

    return body + b"\x90\x01"


def rapdu_get_status(apdu: bytes, state: ConnState) -> bytes:
    with state.lock:
        if state.session_sid is None:
            return b"\x69\x00"

    if len(apdu) < 3:
        return b"\x67\x00"

    p1 = apdu[2]
    status = card_status_byte()

    if p1 == 0x00:
        inner = b"\x80\x01" + bytes([status]) + b"\x04\x02\x10\x00"
        return b"\x63" + bytes([len(inner)]) + inner + b"\x90\x00"

    if 0x01 <= p1 <= 0x0E:
        return b"\x80\x01" + bytes([status]) + b"\x90\x00"

    return b"\x6A\x00"


def rapdu_output(apdu: bytes, state: ConnState) -> bytes:
    with state.lock:
        if state.session_sid is None:
            return b"\x69\x00"

    global last_display_text, idle_display_text

    if len(apdu) < 4:
        return b"\x67\x00"

    p1 = apdu[2]
    p2 = apdu[3]

    data = b""
    if len(apdu) >= 5:
        lc = apdu[4]
        if 5 + lc <= len(apdu):
            data = apdu[5:5 + lc]

    if p1 not in (0x40, 0x60):
        return b"\x6A\x00"

    text_parts = []
    for tag, value in parse_simple_tlvs(data):
        if tag in (b"\x50", b"\x5F\x20"):
            text_parts.append(decode_best_effort_text(value))

    text = "\n".join([t for t in text_parts if t]).strip()

    if p1 == 0x40:
        if p2 == 0x01:
            idle_display_text = text
            print(f"SICCT OUTPUT idle-display gesetzt: {text!r}")
        else:
            last_display_text = text
            print(f"SICCT OUTPUT display: {text!r}")
    elif p1 == 0x60:
        print(f"SICCT OUTPUT printer: {text!r}")

    return b"\x90\x00"


def rapdu_eject_icc(apdu: bytes, state: ConnState) -> bytes:
    with state.lock:
        if state.session_sid is None:
            return b"\x69\x00"

    if len(apdu) < 3:
        return b"\x67\x00"

    fu_number = apdu[2]

    print(f"EJECT ICC für {state.addr} -> Kartenzustand zurücksetzen, Session bleibt offen, Event 85 folgt")
    cleanup_reader_state()
    schedule_slot_removed_event(state, fu_number)

    return b"\x90\x00"


def transmit_icc_apdu(apdu: bytes, state: ConnState) -> bytes:
    with state.lock:
        if state.session_sid is None:
            return b"\x69\x00"

    global current_conn

    with reader_lock:
        if current_conn is None:
            ok = request_icc()
            if not ok or current_conn is None:
                return b"\x64\xA1"

        try:
            print("ICC APDU ->")
            print(hexdump(apdu))
            data, sw1, sw2 = current_conn.transmit(list(apdu))
            rapdu = bytes(data) + bytes([sw1, sw2])
            print("ICC RAPDU <-")
            print(hexdump(rapdu[:256]))
            if len(rapdu) > 256:
                print(f"... gekürzt angezeigt, insgesamt {len(rapdu)} Bytes")
            return rapdu
        except Exception as e:
            print("ICC transmit error:", e)
            disconnect_current()
            return b"\x64\x00"


def handle_apdu(apdu: bytes, state: ConnState) -> bytes:
    if len(apdu) < 4:
        return b"\x67\x00"

    cla = apdu[0]
    ins = apdu[1]

    if cla != 0x80:
        return transmit_icc_apdu(apdu, state)

    if ins == 0x28:
        return rapdu_init_session(apdu, state)

    if ins == 0x29:
        return rapdu_close_session(apdu, state)

    if ins == 0x12:
        return rapdu_request_icc(apdu, state)

    if ins == 0x11:
        return rapdu_reset_ct_icc(apdu, state)

    if ins == 0x13:
        return rapdu_get_status(apdu, state)

    if ins == 0x15:
        return rapdu_eject_icc(apdu, state)

    if ins == 0x17:
        return rapdu_output(apdu, state)

    return b"\x6D\x00"


def handle_client(conn: socket.socket, addr):
    state = ConnState(conn=conn, addr=addr, conn_id=id(conn))
    try:
        print("verbunden von", addr)

        conn.settimeout(None)

        try:
            conn.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
        except Exception:
            pass

        while True:
            data = conn.recv(65535)
            if not data:
                break

            print("empfangen:")
            print(hexdump(data))

            if len(data) < 10:
                print("zu kurzes Paket")
                break

            apdu = data[10:]
            rapdu = handle_apdu(apdu, state)
            resp = build_response(data, rapdu)

            print("sende:")
            print(hexdump(resp[:256]))
            if len(resp) > 256:
                print(f"... gekürzt angezeigt, insgesamt {len(resp)} Bytes")

            conn.sendall(resp)

    except Exception as e:
        print(f"Client-Fehler {addr}: {e}")
    finally:
        cleanup_reader_state()
        close_socket_hard(state)
        print(f"Socket-Ende für {addr}")


def main():
    rs = list_readers_safe()
    if rs:
        print("Verfügbare Reader:")
        for i, r in enumerate(rs):
            print(f"  [{i}] {r}")
    else:
        print("Keine Reader gefunden.")

    selected = find_reader()
    if selected is not None:
        print(f"Gewählter Reader: {selected}")
    else:
        print("Kein passender Reader ausgewählt/gefunden. Server startet trotzdem.")

    print("Hinweis: Wenn der gewünschte Reader oben sichtbar ist, aber nicht gewählt wird,")
    print("dann EXPECTED_READER_SUBSTR auf einen Teil des Reader-Namens setzen.")

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind((HOST, PORT))
        s.listen(20)
        print(f"lausche auf {HOST}:{PORT}")

        while True:
            conn, addr = s.accept()
            t = threading.Thread(target=handle_client, args=(conn, addr), daemon=True)
            t.start()


if __name__ == "__main__":
    main()
'@ | Set-Content -Path .\sicct_server.py -Encoding UTF8

Write-Host ""
Write-Host "===== Reader-Kurztest ====="
.\.venv\Scripts\python.exe -c "from smartcard.System import readers; print(readers())"

Write-Host ""
Write-Host "===== Serverstart ====="
Write-Host "Wenn hier gleich Reader angezeigt werden und 'lausche auf ...' kommt, ist der Python-Teil fertig."
Write-Host ""

.\.venv\Scripts\python.exe .\sicct_server.py

Write-Host ""
Write-Host "===== EXE bauen (danach separat ausführen) ====="
Write-Host ".\.venv\Scripts\pyinstaller.exe --onefile --console --name sicct_server .\sicct_server.py"
Write-Host "Ergebnis: .\dist\sicct_server.exe"









  
</pre>
