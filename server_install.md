# pyenv + pyenv-virtualenv + JupyterHubをインストールする

pythonの計算環境をサーバー上に構築する必要があったので導入してみてる。

pyenv+anaconda環境を作った後に、Jupyterhubを動かす。
想定してる利用状況としては以下のようになる。
- 一般のユーザーには```sudo```権限を与えない
- pythonのversionは```sudo```権限持ちが管理
- anacondaはpyenvを利用して```/opt/pyenv/...```にインストール
- PATHは```/etc/profile```に書き込む
- Jupyterhubの管理には、新たにJupyterhubの管理ユーザとして```jupyterhub```というアカウントを作成する

注意点として(自分メモとして)

- ```python 3.6```は```opencv```にまだ対応していないので、```python 3.5```を利用する

## pyenv + Anacondaインストール

必要なパッケージをインストールする。

```shell
sudo apt-get install git gcc make openssl libssl-dev libbz2-dev libreadline-dev libsqlite3-dev
```

/opt配下にpyenvを設置する。

```shell
cd /opt
git clone git://github.com/yyuu/pyenv.git ./pyenv
mkdir pyenv/{versions,shims}
```

## Jupyterhubのインストール

- 基本、[Jupyterhub公式ドキュメント](https://github.com/jupyterhub/jupyterhub/wiki/Using-sudo-to-run-JupyterHub-without-root-privileges)通りに設定します。

### Jupyterhub管理ユーザーの作成

Jupyterhub管理ユーザーには、ログインshellとパスワードは不要なので、```adduser```ではなく、```useradd```を使用する。

```shell
sudo useradd jupyterhub
```

### sudospawnerの設定

<!---
> you will need the [sudospawner](https://github.com/jupyter/sudospawner) custom Spawner to enable monitoring the single-user servers with sudo

SudoSpawner

> The SudoSpawner enables JupyterHub to spawn single-user servers without being root, by spawning an intermediate process via sudo, which takes actions on behalf of the user.

--->

```shell
sudo pip install sudospawner
```

インストールすると、```visudo```でJupyterhub用の設定を```/etc/sudoeres```で出来るようになる。
ドキュメントによると、

> 1. specify the list of users for whom rhea can spawn servers (JUPYTER_USERS)
> 2. specify the command that rhea can execute on behalf of users (JUPYTER_CMD)
> 3. give rhea permission to run JUPYTER_CMD on behalf of JUPYTER_USERS without entering a password

ということなので、```rhea```を```jupyterhuhb```と読み替えて以下のように追記する。
また、Jupyterhubの利用ユーザーとして、```user_A```, ```user_B```を想定する。

```visudo
# JupyterHub users
Runas_Alias JUPYTER_USERS = user-A, user-B

# the command(s) the Hub can run on behalf of the above users without needing a password
# the exact path may differ, depending on how sudospawner was installed
Cmnd_Alias JUPYTER_CMD = /usr/local/bin/sudospawner

# actually give the Hub user permission to run the above command on behalf
# of the above users without prompting for a password
jupyterhub-manager ALL=(JUPYTER_USERS) NOPASSWD:JUPYTER_CMD
# or with group settings as below
# rhea ALL=(%jupyterhub) NOPASSWD:JUPYTER_CMD
```

### password無しで、sudo

```shell
sudo -u jupyterhub sudo -n -u $USER /usr/local/bin/sudospawner --help
```

として、成功なら

```shell
Usage: /usr/local/bin/sudospawner [OPTIONS]

Options:

  --help                           show this help information
...
```

のように出力される。

### non-rootでPAM認証を有効にする

```shadow```groupを作成する。

```shell
$ sudo groupadd shadow
$ sudo chgrp shadow /etc/shadow
$ sudo chmod g+r /etc/shadow
```

ユーザーを```shadow```gruopに追加する。

```shell
sudo usermod -a -G shadow jupyterhub
```

### PAM認証のテスト

以下のようにして、PAM認証が機能しているかのテストを行う。

```shell
sudo -u jupyterhub python -c "import pamela, getpass; print(pamela.authenticate('$USER', getpass.getpass()))"
```

### JupyterhubのDBなどを置くディレクトリの作成

DBや、configの置き場所を作成する。

```shell
sudo mkdir /etc/jupyterhub
sudo chown jupyterhub /etc/jupyterhub
```

### jupyterhubのconfigを作成する

```shell
jupyterhub --generate-config
```

### 動かしてみる

```shell
cd /etc/jupyterhub
sudo -u jupyterhub jupyterhub --JupyterHub.spawner_class=sudospawner.SudoSpawner
sudo -u jupyterhub /opt/.pyenv/shims/jupyterhub --JupyterHub.spawner_class=sudospawner.SudoSpawner
```

### kernelの追加

初期状態ではjupyterhubを起動しているPythonしか利用できないため、kernelを追加する

```shell
ipython kernel install --name enviroment --user
```

enviromentには追加したいpyenvの環境名が入る