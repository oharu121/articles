## はじめに

Windows 11 の新しい右クリックメニュー、便利なはずが「7-Zip で展開」「VS Code で開く」などがワンアクションで出なくて地味にストレス…という人向けに、レジストリを1か所追加するだけで、旧（クラシック）メニューを既定に戻す手順です。

**再起動は不要**。Explorer（エクスプローラー）だけ再起動します。
※いつでも元の Windows 11 スタイルに戻せます。

以下の記事を参考しました。

https://www.reddit.com/r/sysadmin/comments/1frq94l/guide_restore_old_rightclick_context_menu_in/

## 1\. 旧メニューに戻す

下記をコマンドプロンプトに貼って実行してください。

警告が出ても複数行をペーストして大丈夫です！

```
@echo off
:: Set "Old" Explorer Context Menu as Default
reg add "HKEY_CURRENT_USER\SOFTWARE\CLASSES\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /ve /f

:: Remove Explorer "Command Bar"
reg add "HKCU\Software\Classes\CLSID\{d93ed569-3b3e-4bff-8355-3c44f6a52bb5}\InprocServer32" /f /ve

:: Restart Windows Explorer (no reboot needed)
taskkill /f /im explorer.exe >nul 2>&1
start explorer.exe

:: Done
echo Done. Right-click any file/folder and check the classic menu.
```

**📌 ポイント**

>* No reboot（OS再起動なし）
>* No software（外部ツール不要）

**動作確認の目安**

>* 実行後すぐ、右クリックで旧メニューが出ます
>* 「その他のオプションを表示（Shift+F10）」を押さなくても、7-Zipや「VS Codeで開く」などが最上位に出る

## 2\. Windows 11の新メニューへ戻す

下記をコマンドプロンプトに貼って実行してください。

```
@echo off
:: Restore Win 11 Explorer Context Menu
reg.exe delete "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}" /f

:: Restore Win 11 Explorer Command Bar
reg.exe delete "HKCU\Software\Classes\CLSID\{d93ed569-3b3e-4bff-8355-3c44f6a52bb5}" /f

:: Restart Windows Explorer (no reboot needed)
taskkill /f /im explorer.exe >nul 2>&1
start explorer.exe

:: Done
echo Restored. The Windows 11 context menu is back.
```
