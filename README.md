# acsuwifi-login

信州大学 ACSU の学内 Wi-Fi 認証をコマンドから実行するためのスクリプトです。

学内 Wi-Fi に接続した状態で実行すると、ACSU のログイン処理を行います。

## 対応環境

- Linux
  - Bash
  - curl
  - secret-tool
- Windows
  - PowerShell
  - Windows 標準の DPAPI によるパスワード保存

## 注意

このスクリプトは学内ネットワークに接続した状態で使うことを想定しています。

パスワードをスクリプト内に直接書いてもいいが、一応
Linux では `secret-tool`、Windows では PowerShell の `ConvertFrom-SecureString` を使って保存します。

---

# Linux 版

## 1. 必要パッケージのインストール

Ubuntu / Debian 系では次を実行します。

```bash
sudo apt update
sudo apt install curl libsecret-tools
```

## 2. スクリプト置き場を作成

`~/bin` がなければ作成します。

```bash
mkdir -p ~/bin
```

`~/bin` に PATH を通します。

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

確認します。

```bash
echo "$PATH"
```

## 3. ログインスクリプトを作成

```bash
vi ~/bin/shinshu-login
```

中身は以下です。
`ID="学籍番号"` の部分を自分の学籍番号に変更してください。

```bash
#!/bin/bash

ID="学籍番号"
PASS=$(secret-tool lookup service shinshu-acsu account "$ID")

if [ -z "$PASS" ]; then
    echo "パスワード取得失敗"
    exit 1
fi

RESULT=$(curl -s -X POST \
    -d "uid=$ID" \
    -d "pwd=$PASS" \
    https://login.shinshu-u.ac.jp/cgi-bin/Login.cgi)

echo "$RESULT" | grep -q "Login Success" && echo "接続成功" || echo "接続失敗"
```

実行権限を付与します。

```bash
chmod +x ~/bin/shinshu-login
```

## 4. ACSU パスワードを保存

```bash
secret-tool store --label="Shinshu ACSU Password" service shinshu-acsu account "自分の学籍番号"
```

入力待ちになるので、ACSU のパスワードを入力します。

## 5. パスワード保存確認

```bash
secret-tool lookup service shinshu-acsu account "自分の学籍番号"
```

保存したパスワードが表示されれば OK です。

## 6. 実行

学内 Wi-Fi に接続した状態で実行します。

```bash
shinshu-login
```

絶対パスでも実行できます。

```bash
~/bin/shinshu-login
```

成功時：

```text
接続成功
```

失敗時：

```text
接続失敗
```

---

# Windows 版

Windows では PowerShell を使います。
パスワードは Windows のユーザーアカウントに紐づく形で暗号化保存します。

## 1. スクリプト置き場を作成

PowerShell を開いて、次を実行します。

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\Scripts\shinshu-acsu"
cd "$env:USERPROFILE\Scripts\shinshu-acsu"
```

## 2. パスワード保存用スクリプトを作成

```powershell
notepad .\save-password.ps1
```

以下を保存します。

```powershell
$SecretDir = "$env:USERPROFILE\.shinshu-acsu"
$SecretFile = "$SecretDir\password.txt"

New-Item -ItemType Directory -Force -Path $SecretDir | Out-Null

$SecurePass = Read-Host "ACSU Password" -AsSecureString
$SecurePass | ConvertFrom-SecureString | Set-Content $SecretFile

Write-Host "Password saved"
```

## 3. ログイン用 PowerShell スクリプトを作成

```powershell
notepad .\shinshu-login.ps1
```

以下を保存します。
`$ID = "学籍番号"` の部分を自分の学籍番号に変更してください。

```powershell
$ID = "学籍番号"

$SecretFile = "$env:USERPROFILE\.shinshu-acsu\password.txt"

if (-not (Test-Path $SecretFile)) {
    Write-Host "Password file not found"
    Write-Host "Run save-password.ps1 first"
    exit 1
}

$SecurePass = Get-Content $SecretFile | ConvertTo-SecureString

$BSTR = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecurePass)
$PASS = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($BSTR)
[Runtime.InteropServices.Marshal]::ZeroFreeBSTR($BSTR)

try {
    $Response = Invoke-WebRequest `
        -Uri "https://login.shinshu-u.ac.jp/cgi-bin/Login.cgi" `
        -Method POST `
        -Body @{
            uid = $ID
            pwd = $PASS
        } `
        -UseBasicParsing

    if ($Response.Content -match "Login Success") {
        Write-Host "Login success"
    } else {
        Write-Host "Login failed"
    }
}
catch {
    Write-Host "Network error"
    Write-Host $_.Exception.Message
    exit 1
}
```

## 4. コマンド名 `shinshu-login` で実行できるようにする

```powershell
notepad .\shinshu-login.bat
```

以下を保存します。

```bat
@echo off
powershell -ExecutionPolicy Bypass -File "%USERPROFILE%\Scripts\shinshu-acsu\shinshu-login.ps1"
pause
```

## 5. PATH を通す

PowerShell で次を実行します。

```powershell
$ScriptDir = "$env:USERPROFILE\Scripts\shinshu-acsu"
$CurrentPath = [Environment]::GetEnvironmentVariable("Path", "User")

if ($CurrentPath -notlike "*$ScriptDir*") {
    [Environment]::SetEnvironmentVariable(
        "Path",
        "$CurrentPath;$ScriptDir",
        "User"
    )
    Write-Host "Path added: $ScriptDir"
} else {
    Write-Host "Already in Path"
}
```

PowerShell を一度閉じて、開き直します。

確認します。

```powershell
where.exe shinshu-login
```

次のように表示されれば OK です。

```text
C:\Users\<ユーザー名>\Scripts\shinshu-acsu\shinshu-login.bat
```

## 6. ACSU パスワードを保存

初回だけ実行します。

```powershell
cd "$env:USERPROFILE\Scripts\shinshu-acsu"
powershell -ExecutionPolicy Bypass -File .\save-password.ps1
```

`ACSU Password:` と表示されたら、ACSU のパスワードを入力します。
入力中は文字が表示されません。

## 7. 実行

学内 Wi-Fi に接続した状態で実行します。

```powershell
shinshu-login
```

成功時：

```text
Login success
```

失敗時：

```text
Login failed
```

---


## Windows: 日本語が文字化けする

Windows PowerShell 5.1 では日本語が文字化けする場合があります。Linuxでも環境による。
この README の Windows 版スクリプトでは、表示メッセージを英語にしています。
