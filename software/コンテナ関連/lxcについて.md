# lxc関連の技術

https://knowledge.sakura.ad.jp/2108/

## chroot
プロセスのルートディレクトリを変更することで、プロセスがアクセスできるディレクトリを制限する。
これにより、システムが通常使うライブラリとは異なるものを読み込ませたりできるようになる。
制御の対象はファイル/ディレクトリへのアクセスのみであり、ネットワークやプロセスなどへのアクセスは制御できない。

## jail
FreeBSDに搭載された機能。
ファイルシステムだけでなく、プロセスやデバイスへのアクセス制御が可能。

## cgroups
jailと同じ考え方で実装された、Linuxカーネルの機能。
ファイルシステムの他、CPUリソース、メモリ、デバイスなど様々なリソースへのアクセス制御が可能。
* 特定のプロセスが利用できるメモリ量、CPU時間の制限
* chrootのようにアクセスできるファイル/ディレクトリを制限

# で、LXCとは？
cgrouopsの機能を使って、各種リソースを隔離することで、各コンテナの環境を独立させてる。

## 非特権コンテナ
lxcでは、コンテナ内でroot権限で実行されたプロセスのみ、ホスト環境のroot権限を持つことになってしまう。
そうならないようにした仕組みが、非特権コンテナ。

## liblxc
LXCコンテナを操作するためのライブラリ。



