# NFS演習手順書

## 1. 演習概要

### 演習人数
- 2人

### 用意するもの
- 1人1台 EC2インスタンス

### 演習内容
- 片方が **NFSサーバ**
- 片方が **NFSクライアント**
- クライアントからサーバの共有ディレクトリをマウントする
- `df` で相手の NFS 領域が見えることを確認する
- クライアント側で `touch` でファイルを作成し、サーバ側でそのファイルが確認できれば OK

---

## 2. 役割分担

- **NFSサーバ**：共有ディレクトリを公開する側
- **NFSクライアント**：サーバの共有ディレクトリをマウントする側

### 例
- NFSサーバ：`172.31.27.21`
- NFSクライアント：`172.31.30.158`

---

## 3. なぜプライベートIPを使うのか

NFS は、Web サイトのように外部へ公開するサービスではなく、**決められた相手だけが使う共有フォルダ**です。
そのため、インターネットに公開される **グローバルIP** ではなく、同じ VPC 内で使う **プライベートIP** を使います。

### イメージ
- **グローバルIP**：外の世界に公開されている住所
- **プライベートIP**：社内だけで使う内線番号

NFS は「内輪で使う共有フォルダ」なので、**外向けの住所ではなく内側の住所で接続する**方が安全です。

### 講義での回答例
> NFS はサーバ同士で共有ディレクトリを利用する仕組みで、外部公開する必要がありません。  
> そのため、インターネットに見えるグローバルIPではなく、閉じたネットワーク内で使うプライベートIPを使うことで、安全に通信できるからです。

---

## 4. 事前確認

### 4-1. 自分のIPアドレス確認
サーバ・クライアントの両方で確認します。

```bash
ip a
```

確認するもの：
- サーバの private IP
- クライアントの private IP

---

## 5. セキュリティグループ設定

### 重要
今回の接続は **private IP 同士** で行うため、セキュリティグループでも **相手の private IP**、または **同じセキュリティグループ** を許可対象にします。

**public IP を許可していても、private IP での NFS 通信には一致しないため注意してください。**

### サーバ側EC2
インバウンドルールで以下を許可します。

- タイプ：カスタム TCP
- ポート：`2049`
- ソース：
  - クライアントの private IP `/32`
  - または同じセキュリティグループ

### クライアント側EC2
通常はアウトバウンド全許可のままで問題ありません。
制限している場合は、サーバ宛て `TCP 2049` を許可します。

---

## 6. NFS関連パッケージの確認

Amazon Linux 2023 では、`nfs-utils` が最初から入っている場合があります。
そのため、いきなりインストールするのではなく、まず確認します。

### 6-1. 確認
サーバ・クライアントの両方で実行します。

```bash
rpm -q nfs-utils
```

表示例：

```bash
nfs-utils-2.5.4-2.rc3.amzn2023.0.3.x86_64
```

### 6-2. 未導入の場合のみインストール

```bash
sudo dnf install -y nfs-utils
```

---

## 7. NFSサーバ側の設定

### 7-1. 共有ディレクトリを作成

```bash
sudo mkdir /share
sudo chmod 777 /share
```

今回は演習のため、書き込み確認しやすいように権限を広めに設定しています。

### 7-2. `/etc/exports` を設定

```bash
sudo vi /etc/exports
```

以下を記載します。

```conf
/share 172.31.30.158(rw,no_root_squash)
```

#### 書き方のポイント
- `/share`：共有するディレクトリ
- `172.31.30.158`：接続を許可するクライアントの private IP
- `(rw,no_root_squash)`：読み書き可、root のまま扱う設定

### 注意
**IPアドレスと `(` の間にスペースを入れない**でください。

正しい例：

```conf
/share 172.31.30.158(rw,no_root_squash)
```

### 7-3. 設定を反映

```bash
sudo exportfs -a
```

### 7-4. NFSサーバを起動

```bash
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server
```

### 7-5. 公開状態を確認

```bash
sudo exportfs -v
```

---

## 8. NFSクライアント側の設定

### 8-1. マウントポイント作成

```bash
sudo mkdir /yakiniku
```

### 8-2. `/etc/fstab` を編集

```bash
sudo vi /etc/fstab
```

既存の内容の **下に追記** します。

例：

```fstab
#
UUID=bb1ad377-aefa-4354-a419-b1d6a31d6d2c     /           xfs    defaults,noatime  1   1
UUID=3D07-5F4A        /boot/efi       vfat    defaults,noatime,uid=0,gid=0,umask=0077,shortname=winnt,x-systemd.automount 0 2
172.31.27.21:/share /yakiniku nfs4 defaults 0 0
```

### ポイント
- 既存の UUID の行は消さない
- 一番下に NFS の行を 1 行追加する
- 今回は **教科書準拠で `nfs4`** を使用する

### `0 0` の意味
`/etc/fstab` の右端 2 つの数字です。

- 1つ目の `0`：dump のバックアップ対象にしない
- 2つ目の `0`：起動時に fsck の対象にしない

NFS はローカルディスクではないため、通常 `0 0` にします。

### 8-3. 設定反映

```bash
mount /yakiniku
```

または、`/etc/fstab` 全体をまとめて確認する場合：

```bash
sudo mount -a
```

---

## 9. 動作確認

### 9-1. クライアント側で NFS 領域が見えるか確認

```bash
df -h
```

表示例：

```bash
172.31.27.21:/share
```

のように表示されればマウント成功です。

### 9-2. クライアント側でファイル作成

```bash
touch /yakiniku/test.txt
ls -l /yakiniku
```

### 9-3. サーバ側でファイル確認

```bash
ls -l /share
```

表示例：

```bash
-rw-r--r--. 1 root root 0 Mar 30 15:10 test.txt
```

クライアント側で作成した `test.txt` が、サーバ側の `/share` に見えれば成功です。

### 9-4. 中身まで確認したい場合
クライアント側：

```bash
echo "hello" > /yakiniku/test.txt
```

サーバ側：

```bash
cat /share/test.txt
```

`hello` と表示されれば、同じ領域を見ていることがよりわかりやすいです。

---

## 10. うまくいかないときの確認ポイント

### 10-1. サーバ側で共有ディレクトリが存在するか

```bash
ls -ld /share
```

### 10-2. サーバ側で exports が反映されているか

```bash
cat /etc/exports
sudo exportfs -rav
sudo exportfs -v
```

### 10-3. サーバ側で NFS サービスが起動しているか

```bash
systemctl status nfs-server
```

### 10-4. クライアント側でマウントポイントがあるか

```bash
ls -ld /yakiniku
```

無ければ作成：

```bash
sudo mkdir -p /yakiniku
```

### 10-5. セキュリティグループ確認
- サーバ側で TCP 2049 が許可されているか
- 許可元が **相手の private IP** になっているか
- public IP を許可していないか

---

## 11. 達成条件

以下が確認できれば演習達成です。

### クライアント側

```bash
df -h
```

で相手の NFS 領域が見えること。

### クライアント側

```bash
touch /yakiniku/test.txt
```

### サーバ側

```bash
ls -l /share
```

で `test.txt` が確認できること。

---

## 12. 提出用に短くまとめる場合

NFSサーバ側で共有ディレクトリ `/share` を作成し、`/etc/exports` にクライアントの private IP を指定して公開設定を行った。  
その後 `exportfs -rav` で設定反映し、`nfs-server` を起動した。  
クライアント側では `/etc/fstab` に `172.31.27.21:/share /yakiniku nfs4 defaults 0 0` を追記し、`mount /yakiniku` でマウントした。  
`df -h` で NFS 領域を確認し、クライアント側で作成した `test.txt` がサーバ側の `/share` に存在することを確認できたため、正常に動作していることを確認した。
