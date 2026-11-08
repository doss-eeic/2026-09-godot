#　はじめに


# godotとは


# 目標

## チュートリアルの作成
ゲーム制作をするうえで一つのハードルとなり得るのが、操作方法がわからないというものです。

Godotには、調べればたくさんのテキストチュートリアルが出てくるのですが、godotエディタ上に直接表示されるチュートリアルは存在しません。

今回は、Godotのソースコードをいじることで機能の一つとして、godotエディタ上に直接表示されるチュートリアルを実装しよう、というのがひとつめの目標です。

チュートリアルは、Godotをインストール後の初回の起動時に「はい/いいえ」を選択でき、その後はHelpの部分からいつでもアクセスできるようにすることを想定しました。

## 初期化ログ

### 目的と背景

オブジェクト（ノード）が初期化される順番を、開発中に即座に把握できるようにするため本機能を追加しました。

開発において、新規にクラスを実装する際、既存の別クラスを参照して処理を行うケースが多々あります。この処理を正常に動作させるためには、「参照先の既存クラスの初期化が、新規実装するクラスよりも先に完了している」必要がありますよね。

特にゲーム開発だと、仕様変更が頻発するので先程のケースが発生しやすいんですが、プロジェクト後半で発生すると、オブジェクト数も膨大になっていて初期化の順番をたどるのが難しくなってしまいます。
このような経験から、開発環境の設定をONにするだけで、シーン内のオブジェクトが初期化された順番をログとして手軽に出力・確認できる機能が必要だと考え、実装に至りました。

### 仕様
- スクリプトがアタッチされたオブジェクト（godotではノードと呼ばれる）の初期化時にノード名とスクリプトへのパスを出力する。
- ログ出力の有無をプロジェクトセッティングから変更できるようにする。


## アニメーションのカーソル位置へのキー挿入

### 目的と背景
既存のアニメーション機能において、タイムライン上にキー（Key）を挿入する際、再生位置を示す青いカーソルではなく「マウスポインタの位置」が参照される仕様となっていました。
しかし、マウス操作のみで正確なフレーム位置にキーを打つことは難しく、また初見のユーザーの多くは「青いカーソルの位置にキーが挿入される」と直感的に期待すると考えられます。

そこで、以下の利便性向上を目的として「青いカーソルの位置に挿入」する機能を新規に追加しました。
- 挿入する位置を正確に確定させてからキーを追加したい場合のサポート
- 複数のプロパティに対して、全く同じ位置（フレーム）にキーを揃えて追加したい場合の操作性向上

### 仕様
- タイムライン上でマウスを右クリックした際に表示されるコンテキストメニューにおいて、既存の「Insert Key…」の下に「Insert Key On Cursor」という項目を追加する。
- 「Insert Key On Cursor」を選択すると、現在の青いカーソルの位置（再生位置）にキーが挿入される。


## ズーム設定の環境依存性の解消

### 目的と背景
EditorSettingsの項目に/editors/Panning/2D_Editor_Panning_Schemeがあります。マウスパッドを使用する場合、これはエディター上で画面の拡大縮小操作をピンチ操作にするか２本指スワイプ操作にするか切り替える項目なのですが、macOS上ではこの設定を切り替えてもピンチ操作がズームに割当らたままになってしまうというバグがありました。、これを修正することが目的です。



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
case HELP_TUTORIAL_OVERVIEW:
    _start_tutorial(HELP_TUTORIAL_OVERVIEW);
    break;
case HELP_TUTORIAL_SCENE_AND_NODE:
    _start_tutorial(HELP_TUTORIAL_SCENE_AND_NODE);
    break;
...
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

## 初回起動時のYesを押したときのみ全体コースを開始する
ここでは、初回起動時の確認画面で`Yes`を選んだときに、Overviewだけで終わらず、チュートリアル全体を連続表示する実装を追加しました。実装の中心は`_start_tutorial()`です。

`_tutorial_prompt_confirmed()`では
```cpp
_start_tutorial(HELP_TUTORIAL_OVERVIEW, true);
```
を呼び出し、`full_course = true`のときだけ

- Overview
- Scene and Node Basics
- Adding and Editing Nodes
- Saving and Running a Scene
- Script Basics

の順で`tutorial_steps`へ連結して積むようにしました。

一方で、`Help > Tutorial`からの起動はこれまでどおり各項目ごとに
```cpp
_start_tutorial(HELP_TUTORIAL_..., false)
```
として扱われるため、単体セクションだけが表示されます。

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

また`_update_tutorial_overlay`内に以下の関数コードを追加しました。
```cpp
const int highlight_target_id = _get_tutorial_highlight_target(tutorial_current_entry, tutorial_current_step);
Control *highlight_target = _resolve_tutorial_highlight_target(highlight_target_id);
```

ここでは、現在どのチュートリアル項目が表示されているかを管理する`tutorial_current_entry`と、チュートリアル項目のうち何ステップ目かを管理する`tutorial_current_step`を用いています。`_get_tutorial_highlight_target`はこの二つから切り抜く対象のIDを返し、`_resolve_tutorial_highlight_target`がそのIDを`scene dock`などの実体インスタンスへと変換してくれます。

実体インスタンスを取得しているのは、`get_global_rect()`で画面上の矩形を取得するためです。以上により、`Next`/`Back`の操作に応じて切り抜き部分を切り替えられるようになっています。

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

以上でチュートリアル部分は完成です。

# 機能2:初期化ログ

## 用語整理
この機能の実装過程を説明する上で、ノード、シーン、スクリプトという用語が出てきます。ゲームエンジンを触ったことの無い人からすると初めての概念になると思うので、先に軽く用語を整理しておきます。  
- ノード：ゲームを構成する最小単位の機能   
- シーン：ノード・別のシーンの、再利用可能な集合   
- スクリプト：ノードに付与（アタッチ）し、ノードの機能拡張を行うコード   


ノードの機能の例としては、位置情報や画像表示、効果音、キー/マウス入力の通知などです。
位置情報、画像表示のノードを組み合わせれば、ゲーム内の一つの簡易なキャラクターが作れます。この組み合わせたものがシーンですね。
さて、そのキャラクターの位置情報のノードに、決まった場所を移動させるスクリプトをアタッチすれば、決まったルートを往復するNPCみたいなものが作れます。
完成したNPCキャラクターのシーンは使い回しが可能で、同一シーン内に複数のNPCキャラクターシーンを埋め込むことができます。   
大雑把でいいのでこのあたりの概念を理解しておいてください。

## コード概要

目標としている仕様を実装するためには、ノードの初期化位置と設定項目の指定位置を特定する必要があります。  
opencode(GPT-5.3 Codex. 以降モデル名略)を使用し、以下のように該当箇所を発見しました。  
- SceneState::_instantiate() (scene/resources/packed_scene.cpp)   
- GLOBAL_DEF (core/config/project_settings.h)   
それぞれ詳細を見ていきます。

### ノード初期化位置
SceneState::_instantiate()は一つのメソッドでありながら550行程度の規模で実装されています。おおまかな流れとしては、「ノードの生成→生成に成功したらノードの各種プロパティを設定、以上をシーン内のノードの数だけ繰り返す」となっています。もちろん他の処理も混ざっていますが、初期化ログ機能を実装する上ではここまでの理解で十分でした。  
以下に、コードの流れをコメントとして簡略化し、変更を加えた箇所に焦点を当てたコードを記します。

```cpp
Node *SceneState::_instantiate(GenEditState p_edit_state, LocalVector<DeferredNodePathProperties> *r_parent_deferred_node_paths) const {
	
	/*省略*/

	//コード追記箇所。設定からログ出力の有無のフラグを取得
	const bool log_init_calls = GLOBAL_GET("debug/settings/gdscript/log_init_calls") &&
								(Engine::get_singleton()->is_editor_hint() == false);

	for (int i = 0; i < nc; i++) {		//シーン内の全ノードの数だけ回るfor文

		/*省略*/
		Node *node = nullptr;	//生成されたノードを格納する変数
		/*ノードの生成処理*/

		if (node) {		//ノードの生成に成功したら

			/*ノードのpropertyの設定*/
			
			//コード追記箇所。初期化ログの出力
			if (log_init_calls) {
				String node_name = node->get_name();
				Ref<Script> script = node->get_script();
				String script_path = script.is_valid() ? script->get_path() : String();

				if(script_path.contains("res://")) {
					print_line(node_name + " | " + script_path);	
				}
			}
		}
		/*省略*/
	}	
}
```

 ログを出力する際にノードのシーン内での名称とスクリプトの名称の２つを情報として示したかったので、ノードのプロパティが確定した後にログ出力のコードを記述しています。出力する情報はnode変数から取得しており、get_name()で名称を、get_script()からはnodeにアタッチされているスクリプトを取得可能でした。if(script_path.contains("res://"))は、生成されたノードにスクリプトがアタッチされているかを判別を行っている箇所です。スクリプトは必ず./res以下に配置されるようになっていたので、このような条件判定となっています。  

log_init_callsというフラグは、設定項目でログ出力がonになっており、かつシミュレーション中（作成したシーンを実際に動かしている最中）である場合のみtrueとなるよう記述しています。

### 設定項目の変更
opencodeが発見してくれた「GLOBAL_DEF」はマクロであり、実際に呼び出される関数はグローバル関数の_GLOBAL_DEF(core/config/project_settings.cpp)でした。
_GLOBAL_DEFでは、シングルトンとして実装されているProjectSettingsのインスタンスを取得し、指定された階層にメニューを追加する機能を担っていました。
さて、GLOBAL_DEFは基本的にどこからでも呼び出せるのですが、呼び出している箇所はなるべく固めた方が良いと考え、ノード初期化位置のコードがあった./scene/ディレクトリ以下を対象に以下のコマンドで検索をかけました。
```bash
grep -rn "GLOBAL_DEF" ./scene/
``` 
その結果、./scene/main/scene_tree.cppと./scene/register_scene_types.cppの２つのファイルで主にGLOBAL_DEFが呼び出されていることが判明しました。
今回は./scene/register_scene_types.cpp内に以下のように追加しました。
```cpp
void register_scene_types()
{
	/*省略*/
	for (int i = 0; i < 32; i++) {
		GLOBAL_DEF_BASIC(vformat("%s/layer_%d", PNAME("layer_names/2d_physics"), i + 1), "");
#ifndef NAVIGATION_2D_DISABLED
		GLOBAL_DEF_BASIC(vformat("%s/layer_%d", PNAME("layer_names/2d_navigation"), i + 1), "");
#endif // NAVIGATION_2D_DISABLED
		GLOBAL_DEF_BASIC(vformat("%s/layer_%d", PNAME("layer_names/3d_physics"), i + 1), "");
#ifndef NAVIGATION_3D_DISABLED
		GLOBAL_DEF_BASIC(vformat("%s/layer_%d", PNAME("layer_names/3d_navigation"), i + 1), "");
#endif // NAVIGATION_3D_DISABLED
	}

	for (int i = 0; i < 32; i++) {
		GLOBAL_DEF_BASIC(vformat("%s/layer_%d", PNAME("layer_names/avoidance"), i + 1), "");
	}

	//コード追記箇所
	GLOBAL_DEF("debug/settings/gdscript/log_init_calls", false);

	/*省略*/
}
```
register_scene_types()内の、特にGLOBAL_DEF_BASIC(_GLOBAL_DEFをGLOBAL_DEFとは異なる引数で呼び出すマクロ)が集中している箇所にコードを追記しています（下から4行目）。"debug/settings/gdscript/log_init_calls"は追加する設定の階層を指定しており、falseを渡しているのは、設定のデフォルト値です。


以上で初期化ログ出力機能は完成です。

---

# 機能3: アニメーションのカーソル位置へのキー挿入

## コード概要
本機能の実装にあたり、opencodeにアニメーション関連の修正方針を相談したところ、AI側で具体的なコードの書き換えが行われました。そのため、自動生成された差分コードの処理フローを自身で追跡、及び、実際の動作確認で自動生成されたコードが正しいと判断する形で実装に取り組みました。


処理フローは以下のようになっています。
1. **入力検知:** `AnimationTrackEdit::gui_input` メソッドにてユーザー入力を検知。
2. **メニュー生成:** 入力が右クリックだった場合、`_popup_key_context_menu` が呼び出される。
3. **項目追加:** `_popup_key_context_menu` 内で `PopupMenu` クラスが生成されるため、ここで `PopupMenu::add_icon_item()` を呼び出してボタンを追加（他のボタン生成処理に倣って実装）。
4. **コールバック処理:** 生成されたメニューには `_menu_selected` という関数が登録されており、ユーザーが項目を選択するとこれがコールバックされる。
5. **分岐と実行:** `_menu_selected` 内で受け取った情報（ID）に応じて処理を分岐。再生位置は `AnimationTimeLineEdit` クラスの `play_position_pos` から取得し、既存のキー挿入処理を流用して青いカーソル位置への挿入を実装。

以下に、おおまかなコードの変更箇所を簡略化して記します。

```cpp
// メニュー生成と項目の追加位置
void AnimationTrackEdit::_popup_key_context_menu(...) {
    /*省略*/
    PopupMenu *menu = ... // PopupMenuの生成
	menu->connect(SceneStringName(id_pressed), callable_mp(this, &AnimationTrackEdit::_menu_selected)); //_menu_selectedの結びつけ

    // コード追記箇所: 既存のInsert Key... の上に Insert Key On Cursor を追加
    menu->add_icon_item(get_editor_theme_icon(SNAME("Key")), TTR("Insert Key..."), MENU_KEY_INSERT);
	menu->add_icon_item(get_editor_theme_icon(SNAME("Key")), TTR("Insert Key on Head"), MENU_KEY_INSERT_ON_HEAD);
    /*省略*/
}

```

```cpp

// 選択時のコールバックとキー挿入処理
void AnimationTrackEdit::_menu_selected(int p_index) {
    /*省略*/
    switch (p_index) {
        /*省略*/
        
        case MENU_KEY_INSERT: {
			emit_signal(SNAME("insert_key"), insert_at_pos);	
		} break;
		// コード追記箇所: カーソル位置へのキー挿入処理への分岐
		case MENU_KEY_INSERT_ON_HEAD: {
			emit_signal(SNAME("insert_key"), timeline->get_play_position());
		} break;
    }
}
```


これで機能3は完成です。

# 機能4: ズーム設定の環境依存性の解消

## コード概要

opencodeを使用した結果、`CanvasItemEditor::_gui_input_zoom_or_pan()` メソッドが該当の処理を行っていることが判明したため、ここを起点にデバッガー（gdb）を用いて処理フローの追跡を行いました。コードの流れは以下の通りです。

1. **`_gui_input_view_port()`**: ユーザーからのGUI入力（マウス・キーボード・パッドなど）を一律で受け取る。
2. **`_gui_input_zoom_or_pan()`**: ホイールによる操作か、パン（視点移動）による操作かの大まかな分岐処理を行う。
3. **`ViewPanner::gui_input()`**: 実際の入力内容から、ズームすべきかパンすべきかの具体的な判定を行う。
4. **`_zoom_callback()` / `pan_callback.call()`**: 判定結果に基づき、実際のズーム処理やパン処理を実行する。

## バグの原因

結論からいうと、マウスパッドの入力を処理している箇所に、設定が反映されていないことが原因でした。
下にコードを示しますが、マウス入力の場合に相当するInputEventMouseButtonのブロックでは、if(control_scheme == SCROLL_PANS)のように、設定項目の値を見て処理を分岐させているのに対し、マウスパッド入力の場合に相当するInputEventMagnify/PanGestrueのブロックでは、control_scehemeを見て分岐する処理が無いことがわかります。なので、修正自体は、マウスパッド入力の処理を行っている箇所にcotrol_schemeを見てズームかパンかを切り替える処理を記述すればバグは解消します。


macOSでしかマウスパッド操作のバグが発生しない理由について補足です。環境によって挙動が変わってしまうのは、マウスパッド入力をos側がマウス入力に変換して通知するか、それともマウスパッド入力そのままとして通知するかが分かれているためだと考えられます。その根拠ですが、linux（開発環境参照）、エディタ設定がscroll_zoom、の状態でマウスパッド上のピンチ操作を行った際、is_ctrl_pressed()の値がtrueになっていたことが挙げられます。実際にはctrlキーを押していないわけですから、ピンチ操作がctrlキー＋ホイール回転に変換されて通知されている、ということになるはずです。

```cpp

bool ViewPanner::gui_input(const Ref<InputEvent> &p_event, Rect2 p_canvas_rect) {
	Ref<InputEventMouseButton> mb = p_event;
	if (mb.is_valid()) {								//p_eventがマウスボタンのクリック（ホイールも含む）だった場合

		Vector2 scroll_vec;								//本来はこの行でスクロールの程度をベクトルに格納している	

		if (scroll_vec != Vector2() && mb->is_pressed()) {
			if (control_scheme == SCROLL_PANS) {		//設定がscroll_panのとき
				if (mb->is_ctrl_pressed()) {			//ctrlキーも押されていた場合
					/*省略*/
					zoom_callback.call(zoom, mb->get_position(), p_event);	//ズーム処理の呼び出し
					return true;
				} else {								//ホイールの回転のみの場合
					/*省略*/
					pan_callback.call(-panning * scroll_speed, p_event);	//パン処理の呼び出し
					return true;
				}
			} else {									//設定がscroll_zoomのとき
				/*省略: scroll_panのときとは逆で、ctrlありの場合はパン、なしの場合はズーム処理の呼び出しとなる*/
			}
		}

		/*省略*/
	}

	/*省略*/

	//以下がタッチパッド入力


	//ピンチ操作だった場合
	Ref<InputEventMagnifyGesture> magnify_gesture = p_event;
	if (magnify_gesture.is_valid()) {
		// Zoom gesture
		zoom_callback.call(magnify_gesture->get_factor(), magnify_gesture->get_position(), p_event);
		return true;
	}

	//２本指スワイプ操作だった場合
	Ref<InputEventPanGesture> pan_gesture = p_event;
	if (pan_gesture.is_valid()) {
		if (pan_gesture->is_ctrl_pressed()) {
			zoom_callback.call(zoom, pan_gesture->get_position(), p_event);
			return true;
		}
		pan_callback.call(-pan_gesture->get_delta(), p_event);
	}

	/*省略*/
}

```

## 修正後のコード

```cpp
Ref<InputEventMagnifyGesture> magnify_gesture = p_event;	//ピンチ操作が行われた場合
	if (magnify_gesture.is_valid()) {
		if(control_scheme == SCROLL_PANS){					//設定がscroll_pansの場合
			zoom_callback.call(magnify_gesture->get_factor(), magnify_gesture->get_position(), p_event);	//zoom処理	
		}
		else{　　　											//設定がscroll_zoomの場合
			const float pinch_delta = magnify_gesture->get_factor() - 1.0f;
			if(pinch_delta == 0.f){
				return true;
			}
			pan_callback.call(-Vector2(0, pinch_delta) * scroll_speed * 10, p_event);	//pan処理
		}
		return true;
	}

	Ref<InputEventPanGesture> pan_gesture = p_event;		//２本指スワイプが行われた場合
	if (pan_gesture.is_valid()) {							
		if(control_scheme == SCROLL_PANS){					//設定がscroll_pansの場合
			pan_callback.call(-pan_gesture->get_delta(), p_event);						//pan処理
		}else{												//設定がscroll_zoomの場合

			float pan_zoom_factor = 1.02f;
			float zoom_direction = pan_gesture->get_delta().x - pan_gesture->get_delta().y;
			if (zoom_direction == 0.f) {
				return true;
			}
			float zoom = zoom_direction < 0 ? 1.0 / pan_zoom_factor : pan_zoom_factor;
			zoom_callback.call(zoom, pan_gesture->get_position(), p_event);				//zoom処理
		}
		return true;
	}
```
基本的にはマウスパッド入力のブロックに設定項目を反映した条件分岐を追加しただけになります。入力の程度（どれくらい指を開いたか、どれくらい指をスワイプさせたか）と実際のズーム、パンの程度の変換は既存コードを参考にしつつと実際の操作感からフィードバックしながら調整しました。


これで機能4は完成です。


# おわりに