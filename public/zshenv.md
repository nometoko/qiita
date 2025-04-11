---
title: zshenv vs $PATH vs SSH
tags:
  - Zsh
  - SSH
private: false
updated_at: '2025-03-29T21:42:32+09:00'
id: 0919e7c157e21409bf47
organization_url_name: null
slide: false
ignorePublish: false
---

# 環境

- MacBook Air M2 2022
- Sonoma 14.3.1
- zsh 5.9 (x86_64-apple-darwin23.0)

# 起こったこと

### `ssh`のエラー

ある日、研究室のサーバに ssh しようとすると、以下のようになった。

```shell
% ssh remote_host
zsh:1: command not found: ssh
kex_exchange_identification: Connection closed by remote host
Connection closed by UNKNOWN port 65535
```

`command not found`と出たのでパスが通ってないのかな？と思い、確認したが大丈夫そう。

```shell
% which ssh
/usr/bin/ssh
% print -l $path | grep /usr/bin
/System/Cryptexes/App/usr/bin
/usr/bin
/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin
/Library/Apple/usr/bin
```

# やったこと

`zsh` の設定をいじることがあったので、そのせいかな？と思い、下記を実行。

```shell
% zsh --no-rcs
% ssh remote_host
zsh:1: command not found: ssh
kex_exchange_identification: Connection closed by remote host
Connection closed by UNKNOWN port 65535
```

この時も`which ssh`をすると見つかるし、`$PATH`にも`/usr/bin`は追加されている。
だめだ、、、

ただ、その過程でこのようなエラーに出くわした。

```shell
% zsh
/etc/zshrc:7: command not found: locale
```

そんな訳はないだろうと思い、`which locale`を実行するも、`locale not found`とのこと。
え？じゃあ`$PATH`どうなってるねんということで、確認した結果をいかに示す。

```shell
# 上のコマンドで立ち上げているzshの中で実行
% print -l $path
/opt/local/libexec/gnubin
/usr/local/bin
/opt/local/bin
/opt/local/sbin
/usr/local/lib
/usr/sbin
```

これらはすべて、私の`~/.zshenv`で設定しているものである。
ちなみに、`exit`して元の `zsh` に戻ると`$PATH`は元通りになっている。

## ここまでのまとめ

1. `ssh` を実行すると`command not found`が出るが、パスは通っていそう
2. `zsh --no-rcs`で新たに立ち上げた shell の中でも同じ結果に
3. `zsh`により立ち上げた shell は様子(`$PATH`の設定)がおかしい

# 原因

`~/.zshenv` を以下のように変更していたのが原因だった。

```diff_shell
# 変更前
% cat ~/.zshenv
- export PATH="/opt/local/libexec/gnubin:/usr/local/bin:/opt/local/bin:/opt/local/sbin:/usr/local/lib:/usr/sbin:$PATH"

# 変更後
% cat ~/.zshenv
+ export PATH="/opt/local/libexec/gnubin:/usr/local/bin:/opt/local/bin:/opt/local/sbin:/usr/local/lib:/usr/sbin"
```

差分は最後に`$PATH`をつけるかつけないか、つまり、`$PATH`の設定を追加にするか上書きにするかである。

# 考察

なぜこのようなことがおきたのだろうか？
そもそも、`zshenv`は `zsh` の設定ファイルの中で最初に読まれる設定ファイルのため、上書きしても元の`$PATH`が空なので問題ないはずである。

## `zsh` の設定ファイルと`$PATH`の設定

### 設定ファイルの読み込み順

まず、`zsh`の設定ファイルの読み込み順を確認しておこう。
`man zsh`を見てみると、以下のように書いてあった。

> Commands are first read from /etc/zshenv; this cannot be overridden. \
> [中略] \
> Commands are then read from **$ZDOTDIR/.zshenv**. If the shell is a login shell, commands are read from **/etc/zprofile** and then **$ZDOTDIR/.zprofile**. Then, if the shell is interactive, commands are read from **/etc/zshrc** and then **$ZDOTDIR/.zshrc**. Finally, if the shell is a login shell, **/etc/zlogin** and **$ZDOTDIR/.zlogin** are read. \
> [中略] \
> If **ZDOTDIR** is unset, **HOME** is used instead. Files listed above as being in /etc may be in another directory, depending on the installation.

要するに、`$ZDODIR`が設定されていない私のような環境では、`zsh` を立ち上げた時以下の順で設定ファイルが読み込まれる。

| 順番 | 設定ファイル名  | 備考                                      |
| ---- | --------------- | ----------------------------------------- |
| 1    | `/etc/zshenv`   | 私の環境にはない                          |
| 2    | `~/.zshenv`     |                                           |
| 3    | `/etc/zprofile` | ログインシェルの時のみ                    |
| 4    | `~/.zprofile`   | ログインシェルの時のみ                    |
| 5    | `/etc/.zshrc`   |                                           |
| 6    | `~/.zshrc`      |                                           |
| 7    | `/etc/zlogin`   | ログインシェルの時のみ (私の環境にはない) |
| 8    | `~/.zlogin`     | ログインシェルの時のみ (私の環境にはない) |

### `$PATH`に関する設定

次に、`$PATH`に関する設定を確認しておく。上に示した設定ファイルのうち、`$PATH`に関する設定を行うものを 1 つずつ確認していく。

- `~/.zshenv`
  - 先ほど示したように、主に`/opt`以下のパスを追加している。 (MacPorts のため)
- `/etc/zprofile`
  - `/etc/zprofile`では、`path_helper`というコマンドを呼んでいる。
  - `man path_helper`によると、`path_helper`は、`/etc/paths`と`/etc/paths.d/*`に書かれた内容を元に、`$PATH`を設定する。
  - `/etc/paths`の中身は以下の通り。
    ```
    /usr/local/bin
    /System/Cryptexes/App/usr/bin
    /usr/bin
    /bin
    /usr/sbin
    /sbin
    ```
  - `/etc/paths.d/*`の中身は以下の通り。
    ```
    /var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin
    /var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin
    /var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin
    /opt/X11/bin
    /Library/Apple/usr/bin
    ```
- `~/.zprofile`
  - `~/.zshenv`と同様、MacPorts 関係の設定をしている。

### `zsh` から `zsh` を立ち上げた時の挙動

ここまででこの記事で何を言いたいのか分かってきたと思うが、**`zshenv`を$PATH 上書き方式にした時に**`zsh`を立ち上げた時の挙動を確認しておこう。
まず、ログインシェルを立ち上げた時には、上の 3 つの設定ファイルの`$PATH`が問題なく読み込まれる。
なぜならば、`~/.zshenv`より前に`$PATH`についての設定はされないからである。

:::note info
厳密には、`~/.zshenv`を読む前に`/usr/bin:/bin`は`$PATH`に追加されるようだ。
(`~/.zshenv` の 1 行目に`echo $PATH`を追加して確認した)
また、terminal 特有の`$PATH`の設定も`~/.zshenv`の読み込みの前にされるようだ。
私は kitty を使っているが、その場合、上記に加えて `/Applications/MacPorts/kitty.app/Contents/MacOS:/usr/sbin:/sbin`なども`$PATH`に追加されていた。
:::

しかしながら、ログインシェルから`zsh`を立ち上げた場合、`~/.zshenv`が最初に読み込まれるため、今までの`$PATH`は上書きされてしまう。
`/etc/zprofile`はログインシェルの時のみ読み込まれるためため、`/etc/zprofile`で設定した`/usr/bin`や`/bin`は`$PATH`に追加されない。
そのため、`locale`などのコマンドが見つからないというエラーが出てしまったのである。

## `ssh`の挙動について

次に、`ssh`の挙動について考えてみる。
なぜログインシェルで`ssh`を実行した時に、`command not found`が出たのか。
先程の話から考えると、ログインシェルでは`$PATH`の設定は問題ないはずである。

ここで、**`zshenv`を$PATH 上書き方式にした時の**`ssh`の挙動について以下のことがわかった。
前提として、私が `ssh` しようとしたサーバは、別のサーバを経由して接続するような設定になっている。

```
Host public_host
 User hoge
 HostName public_host.hoge.jp

Host remote_host
 HostName remote_host.hoge.jp
 ProxyCommand ssh -W %h:%p public_host.hoge.jp
```

この時、`public_host`の方には問題なく `ssh` できるのだ。
しかしながら、`remote_host`の方には接続できない。
public_host にも`ssh`はインストールされており、`$PATH`にも`ssh`までのパスは通ってため、public_host の問題によるエラーではなさそうだ。

今までの話を考慮して、`ssh`では踏み台になるサーバを経由して`ssh`する場合に限って、**shell を新たに立ち上げているのではないか**と考えた。

そこで、以下の github を見て`ssh`の実装を確認してみることにした。

https://github.com/openssh/openssh-portable/tree/master

すると、`sshconnect.c`に定義された`ssh_proxy_connect`関数の中に以下のようなコードがあった。

```c
ssh_proxy_connect(struct ssh *ssh, const char *host, const char *host_arg,
    u_short port, const char *proxy_command)
{
    [中略]

	if ((shell = getenv("SHELL")) == NULL || *shell == '\0')
		shell = _PATH_BSHELL;

    [中略]

	/* Fork and execute the proxy command. */
	if ((pid = fork()) == 0) {

        [中略]
		argv[0] = shell;
		argv[1] = "-c";
		argv[2] = command_string;
		argv[3] = NULL;

		/*
		 * Execute the proxy command.  Note that we gave up any
		 * extra privileges above.
		 */
		ssh_signal(SIGPIPE, SIG_DFL);
		execv(argv[0], argv);
		perror(argv[0]);
		exit(1);
	}
  [中略]
}
```

おそらくこの部分で`fork`して新たに shell を`exec`して呼んでいるのだろう。
そのため、ProxyCommand を指定している場合には、`ssh`を実行した時に新たに shell が呼ばれ、上記の`$PATH`の設定が誤った適用されてしまうと考えられる。

# まとめ

- `~/.zshenv`の設定を上書き方式にしてしまったため、ログインシェルでない shell を立ち上げた時に、`$PATH`が上書きされてしまった。
- `ssh`では、踏み台になるサーバを経由して`ssh`する場合に限って、shell を新たに立ち上げているため、その場合には`command not found`が出てしまう。
