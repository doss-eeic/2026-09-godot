#　はじめに


# godotとは


# 目標

## チュートリアルの作成
ゲーム制作をするうえで一つのハードルとなり得るのが、操作方法がわからないというものです。

Godotには、調べればたくさんのテキストチュートリアルが出てくるのですが、godotエディタ上に直接表示されるチュートリアルは存在しません。

今回は、Godotのソースコードをいじることで機能の一つとして、godotエディタ上に直接表示されるチュートリアルを実装しよう、というのがひとつめの目標です。

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
	tutorial_prompt->set_text(TTR("Would you like to start the editor tutorial?\n\nYou can open it later from Help in the top left."));
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

	EditorSettings *editor_settings = EditorSettings::get_singleton();
	const String tutorial_prompt_answered_key = "tutorial/initial_prompt_answered";
	const bool tutorial_prompt_answered = editor_settings->has_setting(tutorial_prompt_answered_key) && bool(editor_settings->get_setting(tutorial_prompt_answered_key));
	if (tutorial_prompt_answered) {
		return;
	}

	if (tutorial_prompt->get_window() != get_last_exclusive_window()) {
		tutorial_prompt->reparent(get_last_exclusive_window());
	}

	tutorial_prompt->reset_size();
	tutorial_prompt->popup_centered();
}
```
`_show_tutorial_prompt()`は上記のような関数であり、無事実行され、`popup_centered()`まで到達することが確認されました。`cmdline_mode == false`は通常のGUIエディタとして動いていることを表します。

その後`conotinue`をし、Godotの画面を開くと以下の画像のようにチュートリアル確認画面が表示されました。これで確認画面の生成については完成です。

## ボタンを押すと次の画面に行く
### 実装
この部分は`シグナル`という要素と、既存のAPIである`connect()`を使い、既存のコードを参考にしながら実装しました。

確認画面生成のコード内の
```cpp
	tutorial_prompt->connect(SceneStringName(confirmed), callable_mp(this, &EditorNode::_tutorial_prompt_confirmed));
```
を例に説明していきます。`connect()`を用いることで、あるボタンをクリックするという`シグナル`(イベント)と、それを受けて実行される関数を接続することができます。

今回の例では、tutorial_promptが(Yesボタンが押されることで生じる)`confirmedシグナル`を発したときに、`_tutorial_prompt_confirmed`が呼ばれるようになります。この関数が実行されることで、Yesボタンとしての機能が果たされるという仕組みです。以後登場する`Next`や`Back`の実装も、基本的な構造は同じです

## 初回起動時にのみ確認画面を表示する
### 実装
今回は、起動をまたいで保存されるフラグが必要になります。ここで着目するのはユーザー単位の設定である`EditorSettings`です。これはGodotエディタを実行した際に、必要に応じて作成/更新されるものとなっており、設定実体は、Linux環境の場合、`editor/settings/editor_settings.*`に保存されます。

まず、`~/.config/godot/editor_settings-4.8.tres`に以下のようなコードを追加しました。
```
tutorial/initial_prompt_answered = false
```

ここで、`Yes/No`ボタンが押された際に呼び出される、`_tutorial_prompt_confirmed()`、`_tutorial_prompt_canceled()`で設定値を書き込み、起動をまたいで保存されるログの生成を行いました。以下では、例として`_tutorial_prompt_confirmed()`のコードを紹介します。
```cpp
void EditorNode::_tutorial_prompt_confirmed() {

	EditorSettings::get_singleton()->set_setting("tutorial/initial_prompt_answered", true);
	EditorSettings::save();
    ...
}
```
チュートリアル確認画面のうちYes(Noを押した場合も同様)を押し、`_tutorial_prompt_confirmed()`が実行されると、`EditorSettings::get_singleton()->set_setting("tutorial/initial_prompt_answered", true);`が実行され、`save()`により、変更内容が保存されます。

ここで先ほどの`~/.config/godot/editor_settings-4.8.tres`のコードを確認すると、以下のように変更が加えられ、期待通りの動作が確認されました。

```
tutorial/initial_prompt_answered = true
```

この変数が`true`となることで、`_show_tutorial_prompt()`内の

```cpp
if (tutorial_prompt_answered) {
	return;
}
```
が実行され、`popup_centered()`が実行される前に関数を抜け出すことで、初回起動時のみに確認画面が表示される機能が実装されました。

## Helpからチュートリアルを再開できるようにする
### コードの特定
今回も、確認画面の作成時と同様に、`editor/`レイヤに絞り特定を行っていくことにしました。

既存のHelpメニューは以下に示す画像のような構成をしています。

始めに、`editor_node.cpp`において、`Report a Bug`で検索をかけました。そこでヒットしたコードの周辺に、以下のようコードを見つけました。
```cpp
help_menu = memnew(PopupMenu);
help_menu->connect(SceneStringName(id_pressed), callable_mp(this, &EditorNode::_menu_option));
_add_to_main_menu(TTRC("Help"), help_menu);
```
これは確認画面において、ボタンが押された際に画面が切り替わる仕組みの部分と似ています。`_menu_option`は以下のように`_menu_option_confirm`を呼び出す関数です。
```cpp
void EditorNode::_menu_option(int p_option) {
	_menu_option_confirm(p_option, false);
}
```
ここがHelpメニューのクリック時の操作に関わるコードとあたりをつけ、`_menu_option`、`_menu_option_confirm`などとともにブレークポイントを置きました。gdbで一連の流れを確認した結果
1. Helpメニュークリック時に`p_option`に項目のIDが運ばれ、`_menu_option`が呼ばれる
2. その中身である`_menu_option_confirm`が呼ばれる
3. `p_option`の値をもとに`_menu_option_confirm`内のswitchのcase分岐により処理関数に移る
4. クリック項目の処理が関数により実行される

という流れでHelpメニューのクリック時の処理が進んでいることがわかりました。`p_option`に渡される値は以下のように、`editor_node.h`内の`MenuOptions`というenumにより定義されています。クリックをすると既存のAPIによって、`PopupMenu`の`id_pressed`シグナルとして`p_option`が渡されるという仕組みになっているようです。
```
// Help menu.
HELP_SEARCH,
HELP_DOCS,
HELP_FORUM,
HELP_COMMUNITY,
HELP_COPY_SYSTEM_INFO,
HELP_REPORT_A_BUG,
HELP_SUGGEST_A_FEATURE,
HELP_SEND_DOCS_FEEDBACK,
HELP_ABOUT,
HELP_SUPPORT_GODOT_DEVELOPMENT,
```

### 実装
はじめに、`MenuOptions`内に以下のコードを追加しました。
```
HELP_TUTORIAL_OVERVIEW,
HELP_TUTORIAL_SCENE_AND_NODE,
HELP_TUTORIAL_ADD_AND_EDIT_NODE,
HELP_TUTORIAL_SAVE_AND_RUN,
HELP_TUTORIAL_SCRIPT_BASICS,
```
これは、`コードの特定`の際に説明したように、どの項目がクリックされたのかを判断するために必要です。

これら各項目がクリックされた際の処理は、以下のように`_menu_option_confirm`にcase文として追加しました。
```cpp
```
クリックされた際には、選択されたcase文が実行されそれぞれの処理に入っていきます。

続いて、以下のメンバーを追加しました。
```
PopupMenu *help_tutorial_menu = nullptr;
```
`PopupMenu`は複数の項目をもつメニューを表すクラスです。これを追加することで、Helpメニューにサブ項目がまとまっているのと同じ処理を用いて、tutorialというメニューの中にそれぞれの項目をまとめることができるようになります。

また、以下のコードを`_build_help_menu`に追加しました。
```cpp
if (!help_tutorial_menu) {
	help_tutorial_menu = memnew(PopupMenu);
	help_tutorial_menu->connect(SceneStringName(id_pressed), callable_mp(this, &EditorNode::_menu_option));
}
help_tutorial_menu->clear(false);
help_tutorial_menu->add_item(TTRC("Editor Screen Overview"), HELP_TUTORIAL_OVERVIEW);
help_tutorial_menu->add_item(TTRC("Scene and Node Basics"), HELP_TUTORIAL_SCENE_AND_NODE);
help_tutorial_menu->add_item(TTRC("Adding and Editing Nodes"), HELP_TUTORIAL_ADD_AND_EDIT_NODE);
help_tutorial_menu->add_item(TTRC("Saving and Running a Scene"), HELP_TUTORIAL_SAVE_AND_RUN);
help_tutorial_menu->add_item(TTRC("Script Basics"), HELP_TUTORIAL_SCRIPT_BASICS);
help_menu->add_submenu_node_item(TTRC("Tutorial"), help_tutorial_menu);
help_menu->add_separator();
```
if文ではすでにチュートリアルメニューが作成されたか否かを判定しています。初回時のみは実行され、それ以降は飛ばされます。その後の
```cpp
help_tutorial_menu->clear(false);
```
により、再構築されるたびに項目の数が増え続けてしまうことを防いでいます。

これらの追加したコードと、既存のAPIにより、Helpメニューにtutorialの各項目を追加することができました。

## 説明文を共通のものにする
ここでは、初回起動時の確認画面において`Yes`を選択した場合の説明文と、Helpメニューのチュートリアルにおける第一項目を選択した場合の説明文を共通のものにするという実装をしました。こちらは`Yes`を押した際に呼ばれる`_tutorial_prompt_confirmed`において、`HELP_TUTORIAL_OVERVIEW`の処理を実行することで、共通した説明文が表示されるようになりました。同様にして、ボタンを押した際に次の説明へと移れるようになっています。

## 画面を暗転させる
### コードの特定
今回の暗転に関わるコードを探す際には、opencodeの力を借りました。`チュートリアル表示中に説明中の場所以外は暗転したいです。それに関わる既存の関数や変数を教えてください。`とプロンプトを打ち、実際に探してもらった結果、以下の関数たちを見つけてくれました。
1. `set_anchors_and_offsets_preset(Control::PRESET_FULL_RECT)`
2. `set_color(Color(r, g, b, a))`
3. `move_to_front()`
4. `ColorRect`

これらの関数、クラスについて説明をしていきます。
1. この関数は、画面全体に新しい領域を自動で配置してくれる関数です。これにより画面全体に、今回であれば暗転させるための画面を自動で配置してくれます。
2. この関数は、文字通り色を配置してくれる関数です。`r`,`g`,`b`がそれぞれ`赤`、`緑`、`青`の色の強さを決定する引数であり、`a`は`透明度`を決定する引数です。いずれの引数も`0.0`から`1.0`の間の値を取ります。
3. この関数は、同じ親の子ノードの中で、指定したノードを前面に持ってきてくれる関数です。暗転がほかのUIに埋もれないようにするために利用します。
4. `ColorRect`はクラスであり、単色の矩形を描画するために使います。gui_baseの子に付け加えることで、`Color()`で指定した内容をもとに画面を表示するための土台となっています。


暗転を実装していくにあたってはこれらのコードを利用していくことになります。

### 実装
最初に、暗転レイヤーの実体を保持するメンバーを以下のように定義しました。
```cpp
ColorRect *tutorial_dimmer = nullptr;
```
続いて、暗転の表示に関する関数`_update_tutorial_overlay`を以下のように書きました。
```cpp
void EditorNode::_update_tutorial_overlay() {
	if (!tutorial_dimmer) {
		return;
	}

	if (tutorial_steps.is_empty()) {
		tutorial_dimmer->hide();
		return;
	}

	tutorial_dimmer->show();
	tutorial_dimmer->move_to_front();
}
```
この関数により、チュートリアルであると判断した際は暗転が表示されるようになります。

次に、暗転の生成ステップを以下のように書きました。
```cpp
tutorial_dimmer = memnew(ColorRect);
gui_base->add_child(tutorial_dimmer);
tutorial_dimmer->set_anchors_and_offsets_preset(Control::PRESET_FULL_RECT);
tutorial_dimmer->set_mouse_filter
tutorial_dimmer->set_color(Color(0, 0, 0, 0.8));
tutorial_dimmer->hide();
```
先ほど紹介した関数たちを使うことで、スクリーンに表示する暗転の生成が行われています。

## 特定の場所の暗転を外す
チュートリアルの際に説明しているボタンなどがどれなのかをわかりやすくするために、暗転を特定の位置から取り除くための実装です。
### コードの特定
暗転の際の`rect`という単語に着目し、UIの座標はどのように取得しているのかを調べたところ、`get_global_rect`という関数が見つかりました。`get_global_rect`は`scene/gui/control.cpp`で以下のように定義されています。
```cpp
Rect2 Control::get_global_rect() const {
	ERR_READ_THREAD_GUARD_V(Rect2());
	Transform2D xform = get_global_transform();
	return Rect2(xform.get_origin(), xform.get_scale() * get_size());
}
```
`Rect2`は`.position(左上の座標)`や`.size(幅と高さ)`などをメンバーとして持つ構造体です。gdbでこの値が呼ばれる流れを確認したところ
1. `Control::_compute_layout_rect(...)`という関数が呼ばれる
2. 上記の関数で計算された位置とサイズがそれぞれ`data.pos_cache`と`data.size_cache`に反映される
3. `get_global_rect`が呼ばれると、`get_global_transform`で反映された内容を読み取り、`Rect2(xform.get_origin(), xform.get_scale() * get_size())`を返す

という一連の流れが確認できました。これにより切り抜くべき場所を特定することができるようになりました。

### 実装
1. 最初にどの切り抜き対象の判別の部分を説明します。

先に実装した暗転の一部を切り抜くということはできないようなので、今回は切り抜く場所の上下左右に4枚の暗転した画面を表示することにしました。今回は主に`scene dock`という項目の切り抜きを例に説明していきます。

全画面を暗転させたときと同様に、上下左右のレイヤーの実体を保持するメンバーを以下のように定義しました。
```
ColorRect *tutorial_dimmer_top = nullptr;
ColorRect *tutorial_dimmer_bottom = nullptr;
ColorRect *tutorial_dimmer_left = nullptr;
ColorRect *tutorial_dimmer_right = nullptr;
```
続いて、切り抜く対象を
```cpp
Control *highlight_target = nullptr;
```
として管理するようにしました。

また、`_update_tutorial_overlay`に以下のコードを追加しました。
```cpp
if (tutorial_current_entry == HELP_TUTORIAL_OVERVIEW) {
	switch (tutorial_current_step) {
		case 1:
			highlight_target = SceneTreeDock::get_singleton();
			break;
		case 2:
			...
	}
}
```
ここでは、現在どのチュートリアル項目が表示されているかを管理する`tutorial_current_entry`と、チュートリアル項目のうち何ステップ目かを管理する`tutorial_current_step`を用いて対象を切り替えています。

ここで`SceneTreeDock::get_singleton()`を使用しているのは、ドックの実体インスタンスを取得して`get_global_rect()`で画面上の矩形を取得するためです。`switch`でステップ番号ごとに対象をきめてあるので、`Next`/`Back`の操作に応じて切り抜き部分を切り替えられるようになっています。

2. 続いて、実際に切り抜きを作る処理を実装しました。

まず初めに、座標の変換を実装しました。`get_global_rect`で取得される座標は、画面全体のうち、その左上を原点としたときの座標です。一方、暗転レイヤー上で配置するときは、暗転レイヤーの左上を原点として考えなければいけないので、対象座標から暗転レイヤーの左上の位置を引いて、基準をそろえる必要があります。この部分は以下のコードで実装をしました。
```cpp
const Vector2 overlay_global_pos = tutorial_dimmer->get_global_position();
Rect2 highlight_rect = highlight_target->get_global_rect();
highlight_rect.position -= overlay_global_pos;
```
その後、切り抜き部分の外側を、上下左右の4つの領域に分割して、それぞれに暗転レイヤーを配置するために以下のコードを追加しました。
```cpp
const float x = highlight_rect.position.x;
const float y = highlight_rect.position.y;
const float w = highlight_rect.size.x;
const float h = highlight_rect.size.y;

set_dimmer_rect(tutorial_dimmer_top, Rect2(0.0f, 0.0f, overlay_size.x, y));
set_dimmer_rect(tutorial_dimmer_bottom, Rect2(0.0f, y + h, overlay_size.x, overlay_size.y - (y + h)));
set_dimmer_rect(tutorial_dimmer_left, Rect2(0.0f, y, x, h));
set_dimmer_rect(tutorial_dimmer_right, Rect2(x + w, y, overlay_size.x - (x + w), h));
```
また、切り抜き部分が表示画面外にはみ出さないために、以下のコードを加えました。
```cpp
const Size2 overlay_size = tutorial_dimmer->get_size();
if (overlay_size.x <= 0.0f || overlay_size.y <= 0.0f) {
	return;
}

highlight_rect.position.x = CLAMP(highlight_rect.position.x, 0.0f, overlay_size.x);
highlight_rect.position.y = CLAMP(highlight_rect.position.y, 0.0f, overlay_size.y);
highlight_rect.size.x = MIN(highlight_rect.size.x, overlay_size.x - highlight_rect.position.x);
highlight_rect.size.y = MIN(highlight_rect.size.y, overlay_size.y - highlight_rect.position.y);
```
以上により、現在説明されているステップの項目だけが暗転しないので、結果として説明部分の強調が実装できました。

# 機能2:初期化ログ


# おわりに