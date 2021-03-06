https://lwn.net/Articles/215996/

ドライバは、メモリ、DMAバッファなど様々なリソースを使用する。
ドライバによっては、バグにより、確保済みリソースの解放が行われない場合がある（ドライバ開発者がそのような処理を書き忘れるなど）。
これにより、システムが不安定になったり、色々弊害が起こる。
このような問題を解決するために作られたのが、device resource managementシステム。

仕組みは簡単。
ドライバがリソースを確保する際、マネジメントシステムがその情報を保持しておき、ドライバが外された時点で、確保済みのリソースが解放するというもの。
リソースをマネジメントシステムの管理対象下に置くためには、既存のリソース確保関数の代わりに、devm_xxx という関数を使う。
例えば、request_irq() の代わりに、[devm_request_irq()](https://elixir.bootlin.com/linux/v4.9.133/source/include/linux/interrupt.h#L170) を使うなど。

devm_xxx()関数は、割り込みだけでなく、DMAバッファ、メモリ確保など、様々なリソースごとに用意されている。
こうして確保されたリソースは、[devres_release_all()](https://elixir.bootlin.com/linux/v4.9.133/source/drivers/base/devres.c#L508)を呼ぶことで、一括解放される。
とはいえ、この関数はドライバが外された時点で自動的に呼ばれるので、明示的に呼ぶ必要はない。

## 使用例
[gpio-keysドライバ](https://elixir.bootlin.com/linux/latest/source/drivers/input/keyboard/gpio_keys.c#L789)で、
devm_kzalloc(), devm_input_allocate_device() 等が呼ばれてる。
