# acsuwifi-login


パスワードはlibsecret-toolsのsecret-toolを使う。使わなくてもいい。

```
sudo apt install libsecret-tools
```

~/binがなければ作ってパスを通す。

```
mkdir -p ~/bin
export PATH="$HOME/bin:$PATH"
source ~/.bashrc
```

IDを自分の学籍番号に変更する。

```
vi ~/bin/shinshu-login
```

忘れず。
```
chmod +x ~/bin/shinshu-login
```

secret-toolのパスワードを設定

```
secret-tool store --label="Shinshu ACSU Password" service shinshu-acsu account "自分の学籍番号"
```
入力待ちになるので、パスワードを入力する。

パスワードを確認する。
```
secret-tool lookup service shinshu-acsu account "学籍番号"
```

学内wifiに接続した状態で実行する。
```
shinshu-login
```

絶対パスでもいい。
```
~/bin/shinshu-login
```

接続成功
接続失敗
が表示される。
