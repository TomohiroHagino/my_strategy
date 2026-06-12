# Blade（ビュー / テンプレート）（Laravel 11）

## ひとことで言うと
Laravel 標準のテンプレートエンジン。`resources/views/*.blade.php` に書き、`{{ }}` などの**簡潔な構文をコンパイルして素のPHP+HTMLに変換**する。コンパイル結果はキャッシュされ高速。

## 役割・なぜ必要か
- コントローラが用意したデータ（`return view('posts.index', ['posts' => $posts])`）を**HTMLに整形して画面を作る**のがビューの仕事。
- 生PHPの `<?php echo ?>` だらけを避け、**自動エスケープ（XSS対策）・レイアウト継承・部品の再利用**を読みやすい構文で提供するためにある。
- ロジック（業務ルール）はモデル/コントローラに置き、ビューは**表示専念**にするのが原則。

## 基本の書き方（コード）
```blade
{{-- resources/views/posts/index.blade.php --}}

{{-- {{ }} は自動でHTMLエスケープされる（XSS安全） --}}
<h1>{{ $title }}</h1>

{{-- {!! !!} は生出力＝エスケープしない（危険・信頼できる値のみ） --}}
<div>{!! $trustedHtml !!}</div>

{{-- 条件分岐 --}}
@if ($posts->isNotEmpty())
    <ul>
    {{-- 繰り返し --}}
    @foreach ($posts as $post)
        <li>{{ $post->title }} / {{ $post->user->name }}</li>
    @endforeach
    </ul>
@else
    <p>投稿がありません</p>
@endif

{{-- @forelse は「ループ or 空」を1構文で --}}
@forelse ($comments as $comment)
    <p>{{ $comment->body }}</p>
@empty
    <p>コメントなし</p>
@endforelse

{{-- 認証・認可・CSRF のショートカット --}}
@auth <span>ようこそ {{ auth()->user()->name }} さん</span> @endauth
@can('update', $post) <a href="...">編集</a> @endcan
<form method="POST" action="/posts">@csrf ... </form>
```
```blade
{{-- レイアウト継承: resources/views/layouts/app.blade.php --}}
<html><body>
  @yield('content')           {{-- 子ビューが流し込む場所 --}}
</body></html>

{{-- 子ビュー側 --}}
@extends('layouts.app')
@section('content')
  <p>ここが content に入る</p>
  @include('posts.partials.card', ['post' => $post])  {{-- 部分テンプレ --}}
@endsection
```

## 実務での使い方・定番パターン
- **エスケープ既定**: `{{ $x }}` は常にエスケープ。ユーザー入力は基本これでXSS安全。`{!! !!}` は CMS本文など「サニタイズ済みの信頼できるHTML」だけ。
- **レイアウト**: 旧来は `@extends/@section/@yield`。Laravel 11 では **Bladeコンポーネントによるレイアウト**（`<x-app-layout>`）も主流。
- **Bladeコンポーネント**で部品化（`php artisan make:component Alert`）。クラス `app/View/Components/Alert.php` + ビュー `resources/views/components/alert.blade.php` が対になり、`<x-alert type="error" />` で呼ぶ。
- **スロット**で中身を差し込む: `<x-alert>{{ $slot }}</x-alert>`、名前付きスロットは `<x-slot:title>`。
- **匿名コンポーネント**（クラス無し・ビューだけ）で軽量な部品も作れる。
- `@props(['type' => 'info'])` でコンポーネントの受け取り引数とデフォルトを宣言。

## ハマりどころ / アンチパターン
- **`{!! !!}` でXSS**: ユーザー入力を生出力すると script 注入される。サニタイズ無しで使わない。→ [security.md](./security.md)
- **ビュー内クエリでN+1**: `@foreach ($posts as $post) {{ $post->user->name }}` でリレーションを都度引くと 1+N クエリ。コントローラ側で `Post::with('user')->get()` する。→ [database.md](./database.md)
- **ビューにロジックを詰める**: `@php ... @endphp` で業務計算・DB更新。責務違反でテスト不能。整形は ViewModel / アクセサ / コンポーネントへ。
- **`@csrf` 忘れ**: POST/PUT/DELETE フォームに付けないと 419 Page Expired。→ [security.md](./security.md)
- **生 `{!! !!}` とエスケープの取り違え**: 「文字化けするから」と安易に `{!! !!}` へ逃げると穴になる。原因は別（多重エスケープ等）を疑う。
- **巨大な1ファイルビュー**: 部品化せず数百行。`@include` / コンポーネントに分割（高凝集・低結合）。

## 関連
[controller.md](./controller.md) / [database.md](./database.md)
