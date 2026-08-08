# ShardSlope
category: crypto
difficult: easy
sources:
```python
# recover.py
#!/usr/bin/env python3
import json
import pathlib
import sys


def inv(value: int, prime: int) -> int:
    return pow(value % prime, -1, prime)


def recover_from_two_shares(share_a: dict, share_b: dict, prime: int) -> int:
    x1, y1 = int(share_a["x"]), int(share_a["y"])
    x2, y2 = int(share_b["x"]), int(share_b["y"])
    if x1 == x2:
        raise ValueError("x-coordinates must be distinct")
    slope = ((y2 - y1) * inv(x2 - x1, prime)) % prime
    return (y1 - slope * x1) % prime


def to_ascii(value: int) -> str:
    size = max(1, (value.bit_length() + 7) // 8)
    return value.to_bytes(size, "big").decode("utf-8", "replace")


def main() -> int:
    if len(sys.argv) != 2:
        print(f"usage: {pathlib.Path(sys.argv[0]).name} escrow_export.json", file=sys.stderr)
        return 2

    export = json.loads(pathlib.Path(sys.argv[1]).read_text())
    prime = int(export["field_prime"])
    shares = export["calibration_batch"]["shares"]
    secret = recover_from_two_shares(shares[0], shares[1], prime)
    print(to_ascii(secret))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```
escrow_export.json
```json
{
  "calibration_batch": {
    "label": "calibration-open-secret",
    "secret_ascii": "NORTHPIER-ESCROW-CALIBRATION-2026",
    "shares": [
      {
        "holder": "ops-audit-01",
        "x": 4,
        "y": 1659485161210225306819544418308223135457098483876820392298702295466554053725073429430791071622558312653514540324964451929040657223256931611637891495584788086
      },
      {
        "holder": "ops-audit-02",
        "x": 13,
        "y": 5393326773933232247163519359501725190235570072599666274970782460266300674606468243412508898058870073310573583060303468704618435640676330009661312839141824006
      }
    ]
  },
  "export_id": "EL-2026-05-16-rcache-17",
  "field_prime": 6864797660130609714981900799081393217269435300143305409394463459185543183397656052122559640661454554977296311391480858037121987999716643812574028291115057151,
  "incident_note": "Calibration and quarantine batches were exported by the same recovery worker after a warm restart. The worker kept rng_cache_id stable for deduplication.",
  "product": "Northpier EscrowLine",
  "quarantine_batch": {
    "label": "quarantine-production-secret",
    "shares": [
      {
        "holder": "quarantine-hsm-02",
        "x": 9,
        "y": 3733841612723006940343974941193502054778471588722845882672080164812399239740262895717726961505675764909877184324973217063907303681357533792839922019461537549
      }
    ],
    "shares_available": 1
  },
  "rng_cache_id": "rng-cache:releng-worker-7:2026-05-16T01:40:12Z",
  "scheme": "Shamir over prime field, degree 1",
  "threshold": 2
}
```
Видим в жсоне подсказку о реюзе нонса и понимаем, что можно вычислить секрет для продакшен секрета с помощью наклона прямой калибровочной кривой.

немного дополняем `recover.py`:
```
# ...
def return_slope(share_a, share_b, prime):
    x1, y1 = int(share_a["x"]), int(share_a["y"])
    x2, y2 = int(share_b["x"]), int(share_b["y"])
    if x1 == x2:
        raise ValueError("x-coordinates must be distinct")
    slope = ((y2 - y1) * inv(x2 - x1, prime)) % prime
    return slope

def decode_secret(slope, prime, secret_share):
    sx, sy = secret_share["x"], secret_share["y"]
    return (sy - slope * sx) % prime
    
# ...

def main() -> int:
	# ...
	secret_share = export["quarantine_batch"]["shares"]
    slope = return_slope(shares[0], shares[1], prime)
    secret2 = decode_secret(slope, prime, secret_share[0])
    print(to_ascii(secret2))
	return 0
```

flag: `caplag{slope_reuse_breaks_2of3_escrow}`

# TagPreview
category: crypto
difficult: easy
sources:
```python
#!/usr/bin/env python3
import argparse
import json
import urllib.request


def get_bytes(url: str) -> bytes:
    with urllib.request.urlopen(url, timeout=5) as response:
        return response.read()


def post_bytes(url: str, body: bytes) -> dict:
    req = urllib.request.Request(
        url,
        data=body,
        method="POST",
        headers={"Content-Type": "application/octet-stream"},
    )
    with urllib.request.urlopen(req, timeout=5) as response:
        return json.loads(response.read().decode())


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("base_url")
    parser.add_argument("--tamper-tag", action="store_true")
    args = parser.parse_args()

    base = args.base_url.rstrip("/")
    record = bytearray(get_bytes(base + "/sealed-note"))
    if args.tamper_tag:
        record[-1] ^= 1
    print(json.dumps(post_bytes(base + "/inspect", bytes(record)), indent=2, ensure_ascii=False))


if __name__ == "__main__":
    main()
```

solution:
```sh
python3 client.py https://task-tagpreview-crypto-easy-3a1cdbdf96250590.caplag-task.ru/ --tamper-tag
```

flag: `caplag{slope_reuse_breaks_2of3_escrow}`

# SealTail
category: crypto
difficult: easy

sources:
```python
# verify_capsules.py
#!/usr/bin/env python3
import hashlib
import pathlib
import struct
import sys


MAGIC = b"STL1"
VERSION = 1
HEADER = struct.Struct(">4sHHII")
SIGNING_CONTEXT = b"SealTail release capsule v1\n"

PUBLIC_N = 0x97F0811C631A43CEB4AA0E04A67FEF8312283C415633890C9D20111746BF3387F356FB79D1CE8108A1ABCB8776C75B68DA106611396B07B0E5E9E6F179A315760F3D6EA7D950F458ED93F13EBC34A01529028BD1E5E1D9FC98F2D83ACDFF028C2EDB9AC1F9F6BA417AFF131957DFD69D842F9EAB40FEAC97E132BC9A57059E8F
PUBLIC_E = 65537

TAGS = {
    0x01: "RELEASE_NOTE",
    0x02: "TELEMETRY",
    0x42: "EXPORT_QUARANTINE_KEY",
}


def parse_capsule(data: bytes):
    if len(data) < HEADER.size:
        raise ValueError("capsule is shorter than header")
    magic, version, _flags, signed_len, sig_len = HEADER.unpack_from(data)
    if magic != MAGIC:
        raise ValueError("bad magic")
    if version != VERSION:
        raise ValueError(f"unsupported version {version}")
    sig_start = HEADER.size
    payload_start = sig_start + sig_len
    if len(data) < payload_start:
        raise ValueError("capsule is shorter than signature")
    signature = data[sig_start:payload_start]
    payload = data[payload_start:]
    if signed_len > len(payload):
        raise ValueError("signed_len exceeds payload length")
    return signed_len, signature, payload


def verify_signature(signed_prefix: bytes, signature: bytes) -> bool:
    if len(signature) != (PUBLIC_N.bit_length() + 7) // 8:
        return False
    actual = pow(int.from_bytes(signature, "big"), PUBLIC_E, PUBLIC_N)
    expected = int.from_bytes(hashlib.sha256(SIGNING_CONTEXT + signed_prefix).digest(), "big")
    return actual == expected


def parse_tlv(payload: bytes):
    offset = 0
    while offset < len(payload):
        if offset + 3 > len(payload):
            raise ValueError(f"truncated TLV header at offset {offset}")
        tag = payload[offset]
        length = int.from_bytes(payload[offset + 1 : offset + 3], "big")
        offset += 3
        if offset + length > len(payload):
            raise ValueError(f"truncated TLV value at offset {offset}")
        yield tag, payload[offset : offset + length]
        offset += length


def main() -> int:
    if len(sys.argv) != 2:
        print(f"usage: {pathlib.Path(sys.argv[0]).name} release.capsule", file=sys.stderr)
        return 2

    data = pathlib.Path(sys.argv[1]).read_bytes()
    signed_len, signature, payload = parse_capsule(data)
    signed_prefix = payload[:signed_len]
    print(f"signature_valid={verify_signature(signed_prefix, signature)}")
    print(f"signed_len={signed_len}")
    print(f"payload_len={len(payload)}")
    print(f"unsigned_tail_len={len(payload) - signed_len}")
    print("records:")
    for index, (tag, value) in enumerate(parse_tlv(payload), 1):
        name = TAGS.get(tag, f"UNKNOWN_0x{tag:02x}")
        preview = value[:80].decode("utf-8", "replace").replace("\n", "\\n")
        print(f"  {index}: tag=0x{tag:02x} {name} length={len(value)} value={preview}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

---

Northpier Robotics использует файлы `*.capsule`, чтобы переносить действия
релиза из сети сборки в карантинный тестовый контур.

Все целые числа записаны в big-endian.

```text
header:
  magic       4 bytes   "STL1"
  version     uint16    1
  flags       uint16    reserved
  signed_len  uint32    number of payload bytes covered by the release signature
  sig_len     uint32    signature length in bytes

body:
  signature   sig_len bytes
  payload     TLV records
```

TLV records:

```text
tag          uint8
length       uint16
value        length bytes
```

Известные теги действий:

```text
0x01 RELEASE_NOTE
0x02 TELEMETRY
0x42 EXPORT_QUARANTINE_KEY
```

Публичный проверяющий включён для инженеров релиза, которым нужно осмотреть
капсулу перед отправкой в очередь.

---

```
gamebun@fedora-main:~$ xxd ~/Downloads/release.capsule    
00000000: 5354 4c31 0001 0000 0000 00b5 0000 0080  STL1............  
00000010: 1ae3 2598 92c1 3726 e8a4 2af2 73b2 c1ef  ..%...7&..*.s...  
00000020: f57d cd9b 29de e61c 8af0 9347 91fb d1e2  .}..)......G....  
00000030: 421b d4d5 b673 9bcd 84ae 1ae7 c1d3 a828  B....s.........(  
00000040: da45 0e6d 0b90 1a8b ba51 eb72 7cf5 aa97  .E.m.....Q.r|...  
00000050: 4c06 8aa1 7919 4197 ad3f f486 e1d7 f679  L...y.A..?.....y  
00000060: a1c0 a6d7 b621 f9f8 344e c7d2 1c82 2a96  .....!..4N....*.  
00000070: 35fc 943a 5c80 c2cd a9c1 1caa 6e0a bc1f  5..:\.......n...  
00000080: e081 9554 f326 c072 a40e 520f 90ae 04e7  ...T.&.r..R.....  
00000090: 0100 6470 726f 6475 6374 3d4e 6f72 7468  ..dproduct=North  
000000a0: 7069 6572 2047 6174 6543 656c 6c0a 7265  pier GateCell.re  
000000b0: 6c65 6173 653d 3230 3236 2e30 352e 3134  lease=2026.05.14  
000000c0: 2d72 6332 0a6c 616e 653d 7175 6172 616e  -rc2.lane=quaran  
000000d0: 7469 6e65 0a6f 7065 7261 746f 723d 7265  tine.operator=re  
000000e0: 6c65 6e67 2d62 6f74 0a73 7461 7475 733d  leng-bot.status=  
000000f0: 7374 6167 6564 0a02 004b 6275 696c 645f  staged...Kbuild_  
00000100: 6964 3d4e 5052 2d47 432d 3839 3134 0a73  id=NPR-GC-8914.s  
00000110: 6361 6e6e 6572 3d6d 696e 6973 6967 6e2d  canner=minisign-  
00000120: 6272 6964 6765 2f31 2e38 2e34 0a70 6f6c  bridge/1.8.4.pol  
00000130: 6963 793d 7369 676e 6564 2d70 7265 6669  icy=signed-prefi  
00000140: 782d 7631 0a                             x-v1.
```

solution:
разбираю `release.capsule`:
```
00000000: 5354 4c31(magic) 0001(version) 0000(flags) 0000 00b5(signed_len) 0000 0080(sig_len)  STL1............
00000010: 1ae3 2598 92c1 3726 e8a4 2af2 73b2 c1ef  ..%...7&..*.s...(///
00000020: f57d cd9b 29de e61c 8af0 9347 91fb d1e2  .}..)......G....
00000030: 421b d4d5 b673 9bcd 84ae 1ae7 c1d3 a828  B....s.........(
00000040: da45 0e6d 0b90 1a8b ba51 eb72 7cf5 aa97  .E.m.....Q.r|...
00000050: 4c06 8aa1 7919 4197 ad3f f486 e1d7 f679  L...y.A..?.....y
00000060: a1c0 a6d7 b621 f9f8 344e c7d2 1c82 2a96  .....!..4N....*.
00000070: 35fc 943a 5c80 c2cd a9c1 1caa 6e0a bc1f  5..:\.......n...
00000080: e081 9554 f326 c072 a40e 520f 90ae 04e7  ...T.&.r..R.....///signature)
00000090: 01(tag)00 64(length)70 726f 6475 6374 3d4e 6f72 7468  ..dproduct=North
000000a0: 7069 6572 2047 6174 6543 656c 6c0a 7265  pier GateCell.re
000000b0: 6c65 6173 653d 3230 3236 2e30 352e 3134  lease=2026.05.14
000000c0: 2d72 6332 0a6c 616e 653d 7175 6172 616e  -rc2.lane=quaran
000000d0: 7469 6e65 0a6f 7065 7261 746f 723d 7265  tine.operator=re
000000e0: 6c65 6e67 2d62 6f74 0a73 7461 7475 733d  leng-bot.status=
000000f0: 7374 6167 6564 0a02 004b 6275 696c 645f  staged...Kbuild_
00000100: 6964 3d4e 5052 2d47 432d 3839 3134 0a73  id=NPR-GC-8914.s
00000110: 6361 6e6e 6572 3d6d 696e 6973 6967 6e2d  canner=minisign-
00000120: 6272 6964 6765 2f31 2e38 2e34 0a70 6f6c  bridge/1.8.4.pol
00000130: 6963 793d 7369 676e 6564 2d70 7265 6669  icy=signed-prefi
00000140: 782d 7631 0a                             x-v1.
```
после изучаем скрипты верификации капсулы и замечаем уязвимые местаЖ
```
# ...
if signed_len > len(payload):
    raise ValueError("signed_len exceeds payload length")
# ...
signed_prefix = payload[:signed_len]
print(f"signature_valid={verify_signature(signed_prefix, signature)}")
# ...
```
проверка реализована неверно и в конец полезной нагрузки можно дописывать произвольные команды, которые никак не повлияют на проверку целостности.
дописываем в конец `420000` и отправляем на серверЖ
```
gamebun@fedora-main:~$ xxd ~/Downloads/release.capsule    
00000000: 5354 4c31 0001 0000 0000 00b5 0000 0080  STL1............  
00000010: 1ae3 2598 92c1 3726 e8a4 2af2 73b2 c1ef  ..%...7&..*.s...  
00000020: f57d cd9b 29de e61c 8af0 9347 91fb d1e2  .}..)......G....  
00000030: 421b d4d5 b673 9bcd 84ae 1ae7 c1d3 a828  B....s.........(  
00000040: da45 0e6d 0b90 1a8b ba51 eb72 7cf5 aa97  .E.m.....Q.r|...  
00000050: 4c06 8aa1 7919 4197 ad3f f486 e1d7 f679  L...y.A..?.....y  
00000060: a1c0 a6d7 b621 f9f8 344e c7d2 1c82 2a96  .....!..4N....*.  
00000070: 35fc 943a 5c80 c2cd a9c1 1caa 6e0a bc1f  5..:\.......n...  
00000080: e081 9554 f326 c072 a40e 520f 90ae 04e7  ...T.&.r..R.....  
00000090: 0100 6470 726f 6475 6374 3d4e 6f72 7468  ..dproduct=North  
000000a0: 7069 6572 2047 6174 6543 656c 6c0a 7265  pier GateCell.re  
000000b0: 6c65 6173 653d 3230 3236 2e30 352e 3134  lease=2026.05.14  
000000c0: 2d72 6332 0a6c 616e 653d 7175 6172 616e  -rc2.lane=quaran  
000000d0: 7469 6e65 0a6f 7065 7261 746f 723d 7265  tine.operator=re  
000000e0: 6c65 6e67 2d62 6f74 0a73 7461 7475 733d  leng-bot.status=  
000000f0: 7374 6167 6564 0a02 004b 6275 696c 645f  staged...Kbuild_  
00000100: 6964 3d4e 5052 2d47 432d 3839 3134 0a73  id=NPR-GC-8914.s  
00000110: 6361 6e6e 6572 3d6d 696e 6973 6967 6e2d  canner=minisign-  
00000120: 6272 6964 6765 2f31 2e38 2e34 0a70 6f6c  bridge/1.8.4.pol  
00000130: 6963 793d 7369 676e 6564 2d70 7265 6669  icy=signed-prefi  
00000140: 782d 7631 0a42 0000                      x-v1.B..
```
```
gamebun@fedora-main:~$ curl -X POST -H "Content-Type: application/octet  
-stream" --data-binary @release.capsule https://task-sealtail-crypto-easy-ade9f3f  
6305514a6.caplag-task.ru/submit  
{"accepted": true, "actions": ["0x01", "0x02", "0x42"], "flag": "caplag{signed_pr  
efix_is_not_a_release}", "note": "quarantine key export approved by release queue  
", "signed_len": 181, "unsigned_tail_len": 3}
```

flag: caplag{signed_prefix_is_not_a_release}

# Эстония
description: 
```
5 февраля 2026 корабль с интересной судьбой остановился в холодном порту, узнайте его историю.

Формат Флага: caplag{Portname_IMO_XXXX0000000_TIR:value}, где IMO –это номер судна, XXXX000000 –номер контейнера, который засветился на кадрах спецназа, value- число-буквенное обозначение TIR у контейнера
```
photo:![[home.png]]
Solution:
По ключевым словам: "Эстония", "Спецназ", "арестовал" находим, что спецназ эстонии арестовал Baltic Spirit под багамским флагом(по датам +- сходится, арестовали 4 февраля, выпустили из порта Мууга 5 февраля).  Ищем в каком порту он стоял и находим Muuga harbour, нахожу снимки со спутника:
![[Pasted image 20260808182840.png]]
и локация полностью подходит под фото, единтсвенное съемка производилась зимой, также сравниваю внешний вид Baltic Spirit с кораблями на фотке и тоже сходится, попали точно в цель. По базе регистраций кораблей вытаскиваю IMO - `9765873`. Дальше ищу видео захвата корабля, нахожу [youtube](https://www.youtube.com/watch?v=3VtMul7deY8). Оттуда вытаскиваю серийный номер, контрольную цифру и первую букву: `S___9696460`. Закидываю в гпт для подбора владельца контейнера с указанием, что это рефрижератор. Получаю 2 варианта SUDU и SEGU, проверяю по базе [BIC](https://app.bic-boxtech.org/search) и выдает информацию только по SEGU9696460. Дальше захожу на сайт и тыкаю по `Unit Enquiry` [seaco](https://seaweb.seacoglobal.com/sap/bc/ui5_ui5/sap/zseaco_ue17/index.html#page|%7B%22id%22%3A%22ContainerDetails%22%7D|0). Ввожу SEGU9696460 и получаю TIR:`GB/C 13362 BV/2016`.
Грац!

flag: Caplag{Muuga_9765873_SEGU9696460_GB/C 13362 BV/2016}