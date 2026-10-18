#　はじめに


# godotとは


# 目標

## チュートリアルの作成
ゲーム制作をするうえで一つのハードルとなり得るのが、操作方法がわからないというものです。

Godotには、調べればたくさんのチュートリアルのようなものが出てくるのですが、説明文や日常的には使われることのない単語がたくさん出てきます。

今回は、Godotのソースコードをいじることで機能の一つとして、上記の問題を解消するチュートリアルを実装しよう、というのがひとつめの目標です。

チュートリアルは、Godotをインストール後の初回の起動時に「はい/いいえ」を選択でき、その後はHelpの部分からいつでもアクセスできるようにすることを想定しました。

## 初期化ログ


# コード全体の概要と構造


# 開発環境


# 機能1:チュートリアル
## 確認画面の生成・初期化プロセス
### コードの特定

今回はエディタに機能を追加していくので、まずは`editor/`レイヤに絞り特定を行っていくことにしました。

最初に、既存の確認画面がどのようなプロセスで生成されているのかを確認しました。<!-- ここにDont saveの画像を貼りたい -->
上記画像のようなUIを想定しているため、`Don't save`と検索をすることでその生成に関わるコードを探すことにしました。検索の結果、`editor/editor_node.cpp:9427`および`editor/editor_node.cpp:9435`が見つかり、その付近のコードを確認することで9426行目に以下の生成コードを発見しました。
```cpp
confirmation = memnew(ConfirmationDialog); 
```
さらに、その近くには、親への追加、サイズ設定、シグナル接続などが並んでいました。
```cpp
confirmation_button = confirmation->add_button(TTRC("Don't Save"), DisplayServer::get_singleton()->get_swap_cancel_ok(), "discard");
gui_base->add_child(confirmation);
confirmation->set_min_size(Vector2(450.0 * EDSCALE, 0));
confirmation->connect(SceneStringName(confirmed), callable_mp(this, &EditorNode::_menu_confirm_current));
confirmation->connect("custom_action", callable_mp(this, &EditorNode::_discard_changes));
confirmation->connect("canceled", callable_mp(this, &EditorNode::_cancel_confirmation));

save_confirmation = memnew(ConfirmationDialog);
```
これらが確認画面生成の一連の処理ではないかとあたりをつけ、ブレークポイントを`editor/editor_node.cpp:9426`に置きました。gdbで一連の流れを確認した結果、
1. 9426行実行前時点では、`confirmation`はヌルポインター
2. 実行後は、`confirmation`はあるアドレス
3. `gui_base->add_child(confirmation);`の前後で`gui_base`の子として`confirmation`が追加された
4. `confirmation->set_min_size(Vector2(450.0 * EDSCALE, 0));`の実行前後で、`get_min_size()`の返す値が変わった
5. 3つの`connect`行の実行後、`save_confirmation`に到達(接続先の関数がボタン操作によって呼ばれるところまでは確認していない)

の流れで確認画面の生成が進んでいることがわかりました。ここまでの流れでは、確認画面を実際に表示する処理までは調べていない状態です。

### 実装
まずは、チュートリアル用の処理を分離するために追加する関数の宣言を行い、それらが既存の処理と干渉しないことを確認しました。

`tutorial_prompt`用の関数
```cpp
_show_tutorial_prompt()  
_tutorial_prompt_confirmed()  
_tutorial_prompt_canceled()
```
次に、既存のコードを参考に、チュートリアルの確認画面の生成、設定、接続を以下のように実装しました。
```cpp
tutorial_prompt = memnew(ConfirmationDialog);
	gui_base->add_child(tutorial_prompt);
	tutorial_prompt->set_title(TTR("Editor Tutorial"));
	tutorial_prompt->set_text(TTR("Would you like to start the editor tutorial?"));
	tutorial_prompt->set_ok_button_text(TTR("Yes"));
	tutorial_prompt->get_cancel_button()->set_text(TTR("No"));
	tutorial_prompt->connect(SceneStringName(confirmed), callable_mp(this, &EditorNode::_tutorial_prompt_confirmed));
	tutorial_prompt->connect("canceled", callable_mp(this, &EditorNode::_tutorial_prompt_canceled));
```
ここで起動時に`_show_tutorial_prompt()`が実行されるかを確認しました。
```cpp
void EditorNode::_show_tutorial_prompt() {
	if (cmdline_mode || !tutorial_prompt) {
		return;
	}

	tutorial_prompt->reset_size();
	tutorial_prompt->popup_centered();
}
```
`_show_tutorial_prompt()`は上記のような関数であり、無事実行され、`popup_centered()`まで到達することが確認されました。`cmdline_mode == false`は通常のGUIエディタとして動いていることを表します。

その後`conotinue`をし、Godotの画面を開くと以下の画像のようにチュートリアル確認画面が表示されました。これで確認画面の生成については完成です。



# 機能2:初期化ログ


# おわりに
