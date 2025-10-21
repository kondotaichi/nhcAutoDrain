# NHC + Slurm 統合チュートリアル: Xidエラー検出と自動ノードドレイン

このチュートリアルでは、Ansibleを使用してNHC (Node Health Check) をSlurmクラスターに統合し、NVIDIA GPUのXidエラーを検出して自動的にノードをドレインする機能を実装する方法を説明します。

## 概要

本チュートリアルでは以下の内容を学習します：

1. **NHCのインストールと設定**: コンテナ環境でのNHC構築
2. **カスタムログ監視機能**: 差分ベースのログ監視モジュール
3. **Xidエラー検出**: NVIDIA GPUの特定Xidエラーの検出
4. **自動ノードドレイン**: エラー検出時のSlurmノード自動ドレイン
5. **動作確認**: 実際のXidエラー注入によるテスト

## 前提条件

- Docker環境でSlurmクラスターが動作していること
- Ansibleがインストールされていること
- NVIDIA GPU搭載ノード（テスト用）

## 1. 環境準備

### インベントリファイルの確認

`inventories/containers.ini`:
```ini
[slurm_master]
slurmctld ansible_connection=community.docker.docker

[slurm_nodes]
c1 ansible_connection=community.docker.docker
c2 ansible_connection=community.docker.docker
c3-highmem ansible_connection=community.docker.docker

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### 必要なAnsibleコレクションのインストール

```bash
ansible-galaxy collection install community.docker
```

## 2. NHCとSlurm統合の実行

### プレイブックの実行

```bash
ansible-playbook -i inventories/containers.ini playbooks/nhc_slurm.yml
```

### 実行内容の詳細

#### 2.1 NHCのインストール

プレイブックは以下の手順でNHCをインストールします：

1. **Python環境の準備**: Python 3.9以上の確保
2. **ビルド依存関係のインストール**: gcc, make, tar等
3. **NHCソースのダウンロード**: GitHubからv1.4.3を取得
4. **コンパイルとインストール**: configure → make → make install

#### 2.2 ロール別NHC設定

**コントローラーノード (slurmctld)**:
```bash
# /etc/nhc/nhc.conf (controller)
source /etc/nhc/scripts/logwatch.nhc

# Container-friendly baseline checks (controller)
* || check_fs_mount_rw /
* || check_ps_daemon slurmctld slurm
* || check_hw_physmem 1k 2TB
* || check_hw_swap 0 2TB
* || check_hw_swap_free 0
* || check_hw_eth eth0
```

**コンピュートノード (slurmd)**:
```bash
# /etc/nhc/nhc.conf (compute)
source /etc/nhc/scripts/logwatch.nhc

# Container-friendly baseline checks (compute)
* || check_fs_mount_rw / 
* || check_ps_daemon slurmd root
* || check_hw_physmem 1k 2TB
* || check_hw_swap 0 2TB
* || check_hw_swap_free 0
* || check_hw_eth eth0
```

#### 2.3 カスタムログ監視モジュール

`/etc/nhc/scripts/logwatch.nhc`に以下の機能を持つモジュールがインストールされます：

- **差分監視**: 前回の読み取り位置を記憶し、追記分のみチェック
- **ローテーション対応**: ファイルのinode変化を検出して先頭から再読み込み
- **エラー回避**: ログファイルが存在しない場合は成功扱い

#### 2.4 Xidエラー検出ルール

特定のNVIDIA Xidエラーを検出するルールが追加されます：

```bash
# /etc/nhc/nhc.conf に追加されるルール
* || check_log_matches /var/log/kern.log  '(^| )Xid (13|48|63|64)'
* || check_log_matches /var/log/syslog    '(^| )Xid (13|48|63|64)'
```

**監視対象のXidエラー**:
- **Xid 13**: Graphics Engine Exception
- **Xid 48**: Double Bit ECC Error
- **Xid 63**: GPU has fallen off the bus
- **Xid 64**: GPU has fallen off the bus (alternate)

#### 2.5 NHCラッパースクリプト

`/usr/local/sbin/nhc_wrapper.sh`が作成され、以下の機能を提供します：

1. **NHCの実行**: ログを`/var/log/nhc/`に保存
2. **エラー検出**: NHCが失敗した場合の処理
3. **自動ドレイン**: `scontrol update`でノードをDRAIN状態に変更

```bash
#!/usr/bin/env bash
set -euo pipefail
LOGDIR=/var/log/nhc
mkdir -p "$LOGDIR"

HOSTNAME="$(hostname -s)"
/usr/sbin/nhc -l "$LOGDIR/${HOSTNAME}.log"
RC=$?

if [[ $RC -ne 0 ]]; then
  REASON="$(tail -n1 "$LOGDIR/${HOSTNAME}.log" 2>/dev/null | sed -e 's/[[:cntrl:]]//g' | cut -c -240)"
  [[ -z "$REASON" ]] && REASON="NHC failure (see ${LOGDIR}/${HOSTNAME}.log)"
  /usr/bin/scontrol update nodename="$HOSTNAME" state=drain reason="$REASON" || true
  exit $RC
fi
```

#### 2.6 Slurm設定の更新

以下の設定が`/etc/slurm/slurm.conf`に追加されます：

```bash
HealthCheckProgram=/usr/local/sbin/nhc_wrapper.sh
HealthCheckInterval=120
HealthCheckNodeState=ANY
```

## 3. Xidエラーテストの実行

### 3.1 テスト用Xidエラーの注入

特定のノード（例：c1）でXidエラーをシミュレートします：

```bash
# ノードc1に接続
docker exec -it c1 bash

# テスト用のXidエラーをログに追加
echo "NVRM: Xid 13: 00/01/00/00: 10de,13c2,1466,1a67, C62, Graphics Engine Exception" >> /var/log/kern.log
echo "NVRM: Xid 48: 00/01/00/00: 10de,13c2,1466,1a67, C62, Double Bit ECC Error" >> /var/log/syslog
```

### 3.2 NHCの手動実行

```bash
# NHCを手動実行してエラー検出を確認
/usr/local/sbin/nhc_wrapper.sh

# 実行結果の確認
echo $?  # 0以外ならエラー検出
```

### 3.3 ノード状態の確認

```bash
# Slurmコントローラーでノード状態を確認
docker exec -it slurmctld scontrol show node c1

# 期待される結果:
# State=IDLE+DRAIN Reason=Test Xid 13 detection
```

### 3.4 ログの確認

```bash
# NHCログの確認
cat /var/log/nhc/c1.log

# システムログの確認
tail -f /var/log/syslog | grep -i xid
```

## 4. 自動ヘルスチェックの確認

Slurmは120秒間隔でヘルスチェックを実行します：
nhc_slurm.yamlで設定を変更できる。

```bash
# ヘルスチェック設定の確認
docker exec -it slurmctld scontrol show config | grep HealthCheck
```

## 5. トラブルシューティング

### 5.1 NHCの実行エラー

```bash
# NHCの詳細実行
/usr/sbin/nhc -v

# 設定ファイルの構文チェック
/usr/sbin/nhc -t
```

### 5.2 ログファイルの権限問題

```bash
# ログディレクトリの権限確認
ls -la /var/log/nhc/
ls -la /var/run/nhc/
```

### 5.3 Slurm設定の再読み込み

```bash
# 設定変更後の再設定
docker exec -it slurm-cluster-slurmctld-1 scontrol reconfigure
```

## 6. 高度な設定

### 6.1 追加のXidエラーの監視

他の重要なXidエラーを追加する場合：

```bash
# /etc/nhc/nhc.confに追加
* || check_log_matches /var/log/kern.log  '(^| )Xid (25|31|43|72|79)'
```

**追加可能なXidエラー**:
- **Xid 25**: ROBUST_CHANNEL_GR_ILLEGAL_NOTIFY
- **Xid 31**: Graphics Engine Exception (alternate)
- **Xid 43**: Graphics Engine Exception
- **Xid 72**: Graphics Engine Exception
- **Xid 79**: Graphics Engine Exception

### 6.2 カスタムヘルスチェックの追加

```bash
# メモリ使用率チェックの例
* || check_cmd_output -m 'MemAvailable.*([0-9]+) kB' cat /proc/meminfo | awk '{if ($2 < 1000000) exit 1}'
```

## 7. まとめ

このチュートリアルでは、以下の内容を実装しました：

1. ✅ **NHCの自動インストール**: Ansibleによる完全自動化
2. ✅ **ロール別設定**: コントローラーとコンピュートノードの最適化
3. ✅ **カスタムログ監視**: 差分ベースの効率的な監視
4. ✅ **Xidエラー検出**: NVIDIA GPUの重要なエラーの自動検出
5. ✅ **自動ノードドレイン**: エラー検出時の自動対応
6. ✅ **動作確認**: 実際のエラー注入によるテスト

この実装により、GPUクラスターの信頼性が大幅に向上し、ハードウェアエラーによるジョブ失敗を事前に防ぐことができます。

## 参考資料

- [NHC公式ドキュメント](https://github.com/mej/nhc)
- [NVIDIA Xid Error Catalog](https://docs.nvidia.com/deploy/xid-errors/index.html)
- [Slurm Health Check Configuration](https://slurm.schedmd.com/slurm.conf.html)
- [Ansible Community Docker Collection](https://docs.ansible.com/ansible/latest/collections/community/docker/)
