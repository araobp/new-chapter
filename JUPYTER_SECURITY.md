# 📁 共通の事前準備：安全な作業フォルダの作成
まず、AI（Claude）を閉じ込めるための専用フォルダをCドライブ直下に作成します。

1. エクスプローラーを開き、Cドライブ直下に **`AI_Workspace`** という名前のフォルダを作成します。
   - パス: `C:\AI_Workspace`
2. **注意**: 分析したいCSVファイルや特許データがある場合は、Boxからこのフォルダの中に**コピー（複製）**して配置してください。

# 🛠️ 対策1：ルートディレクトリを特定フォルダに固定する手順

Jupyterがアクセスできる最上位の階層を `C:\AI_Workspace` に固定し、Boxフォルダがある階層へAIが遡れないようにします。

### 1. 設定ファイルの作成（未作成の場合のみ）
1. コマンドプロンプト（またはPowerShell）を開きます。
2. 以下のコマンドを実行して、Jupyterの設定ファイルを生成します。
   ```bash
   jupyter server --generate-config
   ```
   *(※古いJupyter環境の場合は `jupyter notebook --generate-config` と入力してください)*
3. 画面に設定ファイルが作成されたパスが表示されます。通常は以下の場所に作成されます。
   - `C:\Users\ユーザー名\.jupyter\jupyter_server_config.py`

### 2. 設定ファイルの書き換え
1. 上記のパスにある `jupyter_server_config.py` をメモ帳などのテキストエディタで開きます。
2. ファイルの最下行に、以下の**2行**を追記して保存します。
   ```python
   c.ServerApp.root_dir = 'C:/AI_Workspace'
   c.ServerApp.allow_remote_access = False
   ```
   *(※古いJupyter環境で `jupyter_notebook_config.py` を生成した場合は、`c.NotebookApp.root_dir = 'C:/AI_Workspace'` と記述してください)*

### ✨ 効果
次回以降、Jupyterを起動すると、画面には `C:\AI_Workspace` の中身しか表示されなくなります。Claudeがコード（`os.path` など）を使ってフォルダを上に遡ろうとしても、ここが「最上位のルート」として認識されるため、Boxドライブの存在自体を認知・操作できなくなります。

# 🛡️ 対策2：Python起動時に「削除関数」を無効化する手順

Jupyterのカーネル（Python）が起動した瞬間に、ファイルを削除する関数をシステム内部から強制的に破壊します。

### 1. Startupフォルダを開く
1. キーボードの `Windowsキー + R` を押して「ファイル名を指定して実行」を開きます。
2. `%USERPROFILE%\.ipython\profile_default\startup` と入力して「OK」をクリックします。
   - 該当のフォルダが自動で開きます。

### 2. 無効化スクリプトの配置
1. 開いたフォルダ内で右クリックし、「新規作成」 ＞ 「テキスト ドキュメント」を選択します。
2. ファイル名を **`00-disable-delete.py`** に変更します（拡張子が `.txt` にならないよう注意してください）。
3. そのファイルをメモ帳で開き、以下のコードをそのまま貼り付けて保存します。

```python
import os
import shutil
import subprocess
import sys

# AIによるファイル削除・コマンド実行をブロックする関数
def restricted_action(*args, **kwargs):
    raise PermissionError("【セキュリティ警告】社内ガバナンスにより、AIエージェントによるファイルの削除・変更、およびシステムコマンドの実行は禁止されています。")

# 1. ファイル・フォルダ削除関数の無効化
os.remove = restricted_action
os.unlink = restricted_action
os.rmdir = restricted_action
shutil.rmtree = restricted_action

# 2. 外部コマンド実行（コマンドプロンプトの操作）の無効化
os.system = restricted_action
subprocess.Popen = restricted_action
subprocess.run = restricted_action
subprocess.call = restricted_action
subprocess.check_call = restricted_action
subprocess.check_output = restricted_action

print("🔒 安全対策（ファイル削除・外部コマンドの実行制限）が有効化されました。")
```

### ✨ 効果
Jupyter Notebookで新しいノートブックを開くか、カーネルを起動するたびに、この制限コードが裏側で自動実行されます。
Claudeがセル上で `shutil.rmtree()` を使ってファイルを消そうとしたり、`!rmdir` などのコマンドプロンプトの命令（マジックコマンド含む）を実行しようとした場合、Jupyterが即座に `PermissionError` を吐いて処理を強制遮断します。

# 💡 設定後の確認方法

設定が完了したら、Jupyter Notebookを起動し、新しく作成したセルで以下のテストコードを実行してみてください。

```python
import os
os.remove("test.txt")
```

画面に **`PermissionError: 【セキュリティ警告】...`** と赤文字のエラーが表示されれば、Docker無しの安全対策（檻の構築）は成功です。これでBoxの共有データを安全に守りながら、Claude Coworkに安心してマーケティングや特許のリサーチを任せることができます。
