# もう多言語対応で悩まない！Paraglide × TypeScriptで構築する次世代i18nシステム完全ガイド

## はじめに

SvelteKitアプリの多言語対応で「翻訳キーの管理が大変」「TypeScriptの型チェックが効かない」「複数言語でのルーティングが面倒」なんて頭を抱えたことはありませんか？

従来のi18nライブラリは設定が複雑で、翻訳文字列の型安全性が保証されないし、URLの言語切り替えも手動実装が必要だったりと、面倒な作業が山積みですよね。

「もっとシンプルで、TypeScript完全対応で、SvelteKitとの統合がスムーズなi18nツールはないものか...」

この記事では、**Paraglide × TypeScriptで実現する革新的な多言語システム**の構築方法を実装例とともに徹底解説します！コンパイル時の型チェック、自動翻訳キー生成、パフォーマンス最適化まで、実務で即戦力となる内容をお届けします。

## Paraglideとは何か

Paraglideは、**コンパイル時に翻訳を処理するSvelteKit専用の次世代i18nライブラリ**です。

**💡 Paraglideの革新性**

> * コンパイル時の翻訳処理でランタイムオーバーヘッド０
> * 完全なTypeScript型サポート
> * 未使用翻訳キーの自動検出とTree Shaking
> * SvelteKitルーティングとの完璧な統合

```typescript:project.inlang.json
{
  "$schema": "https://inlang.com/schema/project",
  "sourceLanguageTag": "en",
  "languageTags": ["en", "ja", "ko"],
  "modules": [
    "https://cdn.jsdelivr.net/npm/@inlang/message-lint-rule-empty-pattern@latest/dist/index.js",
    "https://cdn.jsdelivr.net/npm/@inlang/message-lint-rule-missing-translation@latest/dist/index.js"
  ],
  "plugin.inlang.messageFormat": {}
}
```

## 基本的なセットアップ実装

### 1\. プロジェクトの初期化

```bash
npm create svelte@latest paraglide-demo
cd paraglide-demo
npm install
```

### 2\. Paraglideの導入

```bash
npx @inlang/paraglide-js init
npm install @inlang/paraglide-js @inlang/paraglide-sveltekit
```

### 3\. 基本設定の実装

```typescript:vite.config.js
import { paraglide } from '@inlang/paraglide-sveltekit/vite';
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [
    paraglide({
      project: './project.inlang',
      outdir: './src/lib/paraglide'
    }),
    sveltekit()
  ]
});
```

```typescript:src/hooks.client.ts
import { ParaglideJS } from '@inlang/paraglide-sveltekit';
import { i18n } from '$lib/i18n';

ParaglideJS({
  i18n
}).then((paraglide) => {
  paraglide.mount();
});
```

```typescript:src/hooks.server.ts
import { ParaglideJS } from '@inlang/paraglide-sveltekit';
import { i18n } from '$lib/i18n';
import { sequence } from '@sveltejs/kit/hooks';

export const handle = sequence(
  ParaglideJS({
    i18n
  }).handle()
);
```

### 4\. 国際化設定の実装

```typescript:src/lib/i18n.ts
import type { ParaglideLocals } from '@inlang/paraglide-sveltekit';
import { createI18n } from '@inlang/paraglide-sveltekit';
import { match as int } from '../params/int';

export const i18n = createI18n({
  defaultLanguageTag: 'en',
  languageTags: ['en', 'ja', 'ko']
});

declare global {
  namespace App {
    interface Locals extends ParaglideLocals {}
    interface PageData {
      i18n: {
        language: string;
        textDirection: 'ltr' | 'rtl';
      };
    }
  }
}
```

## TypeScript統合と翻訳メッセージ管理

### 1\. メッセージファイルの作成

```json:messages/en.json
{
  "hello": "Hello, World!",
  "welcome": "Welcome to {name}!",
  "navigation": {
    "home": "Home",
    "about": "About",
    "contact": "Contact",
    "profile": "Profile"
  },
  "form": {
    "submit": "Submit",
    "cancel": "Cancel",
    "required": "This field is required",
    "email": {
      "label": "Email Address",
      "placeholder": "Enter your email",
      "invalid": "Please enter a valid email address"
    },
    "password": {
      "label": "Password",
      "placeholder": "Enter your password",
      "minLength": "Password must be at least {min} characters"
    }
  },
  "errors": {
    "network": "Network connection failed",
    "notFound": "Page not found",
    "serverError": "Internal server error occurred"
  },
  "datetime": {
    "now": "Now",
    "today": "Today",
    "yesterday": "Yesterday",
    "lastWeek": "Last week"
  }
}
```

```json:messages/ja.json
{
  "hello": "こんにちは、世界！",
  "welcome": "{name}へようこそ！",
  "navigation": {
    "home": "ホーム",
    "about": "概要",
    "contact": "お問い合わせ",
    "profile": "プロフィール"
  },
  "form": {
    "submit": "送信",
    "cancel": "キャンセル",
    "required": "この項目は必須です",
    "email": {
      "label": "メールアドレス",
      "placeholder": "メールアドレスを入力",
      "invalid": "有効なメールアドレスを入力してください"
    },
    "password": {
      "label": "パスワード",
      "placeholder": "パスワードを入力",
      "minLength": "パスワードは{min}文字以上で入力してください"
    }
  },
  "errors": {
    "network": "ネットワーク接続に失敗しました",
    "notFound": "ページが見つかりません",
    "serverError": "内部サーバーエラーが発生しました"
  },
  "datetime": {
    "now": "今",
    "today": "今日",
    "yesterday": "昨日",
    "lastWeek": "先週"
  }
}
```

### 2\. 型安全な翻訳ユーティリティの実装

```typescript:src/lib/i18n/utils.ts
import * as m from '$lib/paraglide/messages';
import { languageTag } from '$lib/paraglide/runtime';
import type { AvailableLanguageTag } from '$lib/paraglide/runtime';

export interface TranslationOptions {
  [key: string]: string | number;
}

export class I18nManager {
  static getCurrentLanguage(): AvailableLanguageTag {
    return languageTag();
  }
  
  static isRTL(lang: AvailableLanguageTag = languageTag()): boolean {
    const rtlLanguages: AvailableLanguageTag[] = ['ar', 'he', 'fa'];
    return rtlLanguages.includes(lang);
  }
  
  static formatMessage(
    messageKey: keyof typeof m, 
    params?: TranslationOptions
  ): string {
    const message = m[messageKey];
    if (typeof message === 'function' && params) {
      return message(params);
    }
    return typeof message === 'string' ? message : String(message);
  }
  
  static getLanguageNativeName(lang: AvailableLanguageTag): string {
    const nativeNames: Record<AvailableLanguageTag, string> = {
      'en': 'English',
      'ja': '日本語',
      'ko': '한국어'
    };
    return nativeNames[lang] || lang;
  }
  
  static formatNumber(
    num: number, 
    options?: Intl.NumberFormatOptions,
    lang: AvailableLanguageTag = languageTag()
  ): string {
    return new Intl.NumberFormat(lang, options).format(num);
  }
  
  static formatDate(
    date: Date, 
    options?: Intl.DateTimeFormatOptions,
    lang: AvailableLanguageTag = languageTag()
  ): string {
    return new Intl.DateTimeFormat(lang, options).format(date);
  }
  
  static formatRelativeTime(
    date: Date,
    lang: AvailableLanguageTag = languageTag()
  ): string {
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffMinutes = Math.floor(diffMs / (1000 * 60));
    const diffHours = Math.floor(diffMs / (1000 * 60 * 60));
    const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));
    
    const rtf = new Intl.RelativeTimeFormat(lang, { numeric: 'auto' });
    
    if (Math.abs(diffMinutes) < 60) {
      return rtf.format(-diffMinutes, 'minute');
    } else if (Math.abs(diffHours) < 24) {
      return rtf.format(-diffHours, 'hour');
    } else {
      return rtf.format(-diffDays, 'day');
    }
  }
}

export const i18nUtils = I18nManager;
```

## コンポーネント実装例

### 1\. 言語切り替えコンポーネント

```svelte:src/lib/components/LanguageSwitcher.svelte
<script lang="ts">
  import { page } from '$app/stores';
  import { replaceLanguageInUrl, availableLanguageTags } from '$lib/paraglide/utils';
  import { languageTag } from '$lib/paraglide/runtime';
  import { i18nUtils } from '$lib/i18n/utils';
  import * as m from '$lib/paraglide/messages';
  
  let isOpen = false;
  
  function switchLanguage(newLang: string) {
    const newUrl = replaceLanguageInUrl($page.url, newLang);
    window.location.href = newUrl.href;
    isOpen = false;
  }
  
  function toggleDropdown() {
    isOpen = !isOpen;
  }
  
  function handleOutsideClick(event: MouseEvent) {
    const target = event.target as Element;
    if (!target.closest('.language-switcher')) {
      isOpen = false;
    }
  }
</script>

<svelte:window on:click={handleOutsideClick} />

<div class="language-switcher" class:open={isOpen}>
  <button 
    class="current-language" 
    on:click={toggleDropdown}
    aria-label={m.switchLanguage()}
  >
    <span class="flag">
      {#if languageTag() === 'en'}🇺🇸
      {:else if languageTag() === 'ja'}🇯🇵
      {:else if languageTag() === 'ko'}🇰🇷
      {/if}
    </span>
    <span class="name">
      {i18nUtils.getLanguageNativeName(languageTag())}
    </span>
    <svg class="chevron" class:rotated={isOpen} viewBox="0 0 20 20" fill="currentColor">
      <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
    </svg>
  </button>
  
  {#if isOpen}
    <div class="dropdown">
      {#each availableLanguageTags as lang}
        {#if lang !== languageTag()}
          <button 
            class="language-option"
            on:click={() => switchLanguage(lang)}
          >
            <span class="flag">
              {#if lang === 'en'}🇺🇸
              {:else if lang === 'ja'}🇯🇵
              {:else if lang === 'ko'}🇰🇷
              {/if}
            </span>
            <span class="name">
              {i18nUtils.getLanguageNativeName(lang)}
            </span>
          </button>
        {/if}
      {/each}
    </div>
  {/if}
</div>

<style>
  .language-switcher {
    position: relative;
    display: inline-block;
  }
  
  .current-language {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    background: white;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem;
    cursor: pointer;
    font-size: 0.875rem;
    transition: all 0.2s;
  }
  
  .current-language:hover {
    background: #f9fafb;
    border-color: #9ca3af;
  }
  
  .flag {
    font-size: 1.125rem;
  }
  
  .chevron {
    width: 1rem;
    height: 1rem;
    transition: transform 0.2s;
  }
  
  .chevron.rotated {
    transform: rotate(180deg);
  }
  
  .dropdown {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
    z-index: 50;
    margin-top: 0.25rem;
  }
  
  .language-option {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    width: 100%;
    padding: 0.75rem 1rem;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 0.875rem;
    transition: background-color 0.2s;
  }
  
  .language-option:hover {
    background: #f3f4f6;
  }
  
  .language-option:first-child {
    border-radius: 0.375rem 0.375rem 0 0;
  }
  
  .language-option:last-child {
    border-radius: 0 0 0.375rem 0.375rem;
  }
</style>
```

### 2\. 多言語対応フォームコンポーネント

```svelte:src/lib/components/ContactForm.svelte
<script lang="ts">
  import { enhance } from '$app/forms';
  import * as m from '$lib/paraglide/messages';
  import { i18nUtils } from '$lib/i18n/utils';
  import { languageTag } from '$lib/paraglide/runtime';
  
  interface FormData {
    name: string;
    email: string;
    message: string;
  }
  
  interface FormErrors {
    name?: string;
    email?: string;
    message?: string;
    general?: string;
  }
  
  let form: FormData = {
    name: '',
    email: '',
    message: ''
  };
  
  let errors: FormErrors = {};
  let isSubmitting = false;
  let isSuccess = false;
  
  function validateEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
  
  function validateForm(): boolean {
    errors = {};
    let isValid = true;
    
    if (!form.name.trim()) {
      errors.name = m.form_required();
      isValid = false;
    }
    
    if (!form.email.trim()) {
      errors.email = m.form_required();
      isValid = false;
    } else if (!validateEmail(form.email)) {
      errors.email = m.form_email_invalid();
      isValid = false;
    }
    
    if (!form.message.trim()) {
      errors.message = m.form_required();
      isValid = false;
    }
    
    return isValid;
  }
  
  function handleSubmit() {
    if (!validateForm()) {
      return;
    }
    
    isSubmitting = true;
  }
</script>

<form 
  method="POST" 
  action="?/contact"
  class="contact-form"
  class:rtl={i18nUtils.isRTL()}
  use:enhance={() => {
    handleSubmit();
    return ({ result }) => {
      isSubmitting = false;
      if (result.type === 'success') {
        isSuccess = true;
        form = { name: '', email: '', message: '' };
      } else if (result.type === 'failure') {
        errors = result.data?.errors || { general: m.errors_serverError() };
      }
    };
  }}
>
  <div class="form-group">
    <label for="name">{m.form_name_label()}</label>
    <input 
      id="name"
      name="name"
      type="text"
      bind:value={form.name}
      placeholder={m.form_name_placeholder()}
      class:error={errors.name}
      required
    />
    {#if errors.name}
      <span class="error-message">{errors.name}</span>
    {/if}
  </div>
  
  <div class="form-group">
    <label for="email">{m.form_email_label()}</label>
    <input 
      id="email"
      name="email"
      type="email"
      bind:value={form.email}
      placeholder={m.form_email_placeholder()}
      class:error={errors.email}
      required
    />
    {#if errors.email}
      <span class="error-message">{errors.email}</span>
    {/if}
  </div>
  
  <div class="form-group">
    <label for="message">{m.form_message_label()}</label>
    <textarea 
      id="message"
      name="message"
      bind:value={form.message}
      placeholder={m.form_message_placeholder()}
      class:error={errors.message}
      rows="5"
      required
    ></textarea>
    {#if errors.message}
      <span class="error-message">{errors.message}</span>
    {/if}
  </div>
  
  {#if errors.general}
    <div class="general-error">
      {errors.general}
    </div>
  {/if}
  
  {#if isSuccess}
    <div class="success-message">
      {m.form_success()}
    </div>
  {/if}
  
  <div class="form-actions">
    <button 
      type="submit" 
      disabled={isSubmitting}
      class="submit-button"
    >
      {#if isSubmitting}
        <div class="spinner"></div>
        {m.form_submitting()}
      {:else}
        {m.form_submit()}
      {/if}
    </button>
  </div>
</form>

<style>
  .contact-form {
    max-width: 600px;
    margin: 0 auto;
    padding: 2rem;
  }
  
  .contact-form.rtl {
    direction: rtl;
  }
  
  .form-group {
    margin-bottom: 1.5rem;
  }
  
  label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: 500;
    color: #374151;
  }
  
  input, textarea {
    width: 100%;
    padding: 0.75rem;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem;
    font-size: 1rem;
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  
  input:focus, textarea:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }
  
  input.error, textarea.error {
    border-color: #ef4444;
  }
  
  .error-message {
    display: block;
    margin-top: 0.25rem;
    font-size: 0.875rem;
    color: #ef4444;
  }
  
  .general-error {
    padding: 0.75rem;
    background: #fef2f2;
    border: 1px solid #fecaca;
    border-radius: 0.375rem;
    color: #dc2626;
    margin-bottom: 1rem;
  }
  
  .success-message {
    padding: 0.75rem;
    background: #f0fdf4;
    border: 1px solid #bbf7d0;
    border-radius: 0.375rem;
    color: #166534;
    margin-bottom: 1rem;
  }
  
  .form-actions {
    display: flex;
    justify-content: flex-end;
  }
  
  .submit-button {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.75rem 2rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 0.375rem;
    font-size: 1rem;
    font-weight: 500;
    cursor: pointer;
    transition: background-color 0.2s;
  }
  
  .submit-button:hover:not(:disabled) {
    background: #2563eb;
  }
  
  .submit-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
  
  .spinner {
    width: 1rem;
    height: 1rem;
    border: 2px solid transparent;
    border-top: 2px solid currentColor;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  
  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }
</style>
```

## TypeScriptでの高度な活用例

### 1\. 型安全な翻訳キー管理システム

```typescript:src/lib/i18n/types.ts
import type { AvailableLanguageTag } from '$lib/paraglide/runtime';

export interface TranslationContext {
  language: AvailableLanguageTag;
  namespace?: string;
  fallback?: AvailableLanguageTag;
}

export interface PluralRules {
  zero?: string;
  one: string;
  two?: string;
  few?: string;
  many?: string;
  other: string;
}

export interface MessageMetadata {
  key: string;
  description?: string;
  context?: string;
  maxLength?: number;
  examples?: Record<string, string>;
}

export type MessageValidator<T = any> = (value: T, context: TranslationContext) => boolean | string;

export interface TypedTranslation<T = any> {
  key: string;
  validate?: MessageValidator<T>;
  transform?: (value: T) => T;
  metadata?: MessageMetadata;
}
```

```typescript:src/lib/i18n/advanced.ts
import * as m from '$lib/paraglide/messages';
import { languageTag, type AvailableLanguageTag } from '$lib/paraglide/runtime';
import type { TranslationContext, TypedTranslation, PluralRules } from './types';

export class AdvancedI18n {
  private static _instance: AdvancedI18n;
  private _cache = new Map<string, string>();
  
  static getInstance(): AdvancedI18n {
    if (!AdvancedI18n._instance) {
      AdvancedI18n._instance = new AdvancedI18n();
    }
    return AdvancedI18n._instance;
  }
  
  /**
   * 複数形対応の翻訳
   */
  plural(
    count: number, 
    rules: PluralRules, 
    lang: AvailableLanguageTag = languageTag()
  ): string {
    const cacheKey = `plural_${count}_${lang}_${JSON.stringify(rules)}`;
    if (this._cache.has(cacheKey)) {
      return this._cache.get(cacheKey)!;
    }
    
    const pluralRule = new Intl.PluralRules(lang);
    const rule = pluralRule.select(count);
    
    let result: string;
    switch (rule) {
      case 'zero':
        result = rules.zero || rules.other;
        break;
      case 'one':
        result = rules.one;
        break;
      case 'two':
        result = rules.two || rules.other;
        break;
      case 'few':
        result = rules.few || rules.other;
        break;
      case 'many':
        result = rules.many || rules.other;
        break;
      default:
        result = rules.other;
    }
    
    // カウント値を置換
    result = result.replace('{count}', count.toString());
    
    this._cache.set(cacheKey, result);
    return result;
  }
  
  /**
   * リッチテキスト翻訳（HTMLタグを含む）
   */
  rich(
    messageKey: keyof typeof m,
    replacements: Record<string, string | number> = {},
    allowedTags: string[] = ['strong', 'em', 'a']
  ): string {
    const message = m[messageKey] as any;
    let text = typeof message === 'function' ? message(replacements) : String(message);
    
    // HTMLエスケープ（allowedTags以外）
    text = this.escapeHtmlExcept(text, allowedTags);
    
    return text;
  }
  
  /**
   * 条件付き翻訳
   */
  conditional(
    condition: boolean | ((context: TranslationContext) => boolean),
    trueKey: keyof typeof m,
    falseKey: keyof typeof m,
    params: Record<string, any> = {}
  ): string {
    const context: TranslationContext = { language: languageTag() };
    const shouldUseTrueKey = typeof condition === 'function' 
      ? condition(context) 
      : condition;
    
    const messageKey = shouldUseTrueKey ? trueKey : falseKey;
    const message = m[messageKey] as any;
    
    return typeof message === 'function' ? message(params) : String(message);
  }
  
  /**
   * 型安全な翻訳バリデーション
   */
  validateTranslation<T>(
    translation: TypedTranslation<T>, 
    value: T
  ): { isValid: boolean; error?: string } {
    if (!translation.validate) {
      return { isValid: true };
    }
    
    const context: TranslationContext = { language: languageTag() };
    const result = translation.validate(value, context);
    
    if (typeof result === 'boolean') {
      return { isValid: result };
    }
    
    return { isValid: false, error: result };
  }
  
  /**
   * 動的翻訳キー解決
   */
  resolveDynamic(
    keyPattern: string,
    variables: Record<string, string | number>,
    fallback?: string
  ): string {
    try {
      // パターンから実際のキーを生成
      let resolvedKey = keyPattern;
      Object.entries(variables).forEach(([key, value]) => {
        resolvedKey = resolvedKey.replace(`{${key}}`, String(value));
      });
      
      // メッセージオブジェクトから動的にキーを解決
      const message = this.getNestedMessage(m, resolvedKey);
      if (message) {
        return typeof message === 'function' ? message(variables) : String(message);
      }
      
      return fallback || `[Missing: ${resolvedKey}]`;
    } catch (error) {
      console.warn('Failed to resolve dynamic translation:', keyPattern, error);
      return fallback || `[Error: ${keyPattern}]`;
    }
  }
  
  /**
   * パフォーマンス最適化：翻訳の遅延読み込み
   */
  async loadTranslationBundle(
    bundleName: string,
    lang: AvailableLanguageTag = languageTag()
  ): Promise<Record<string, any>> {
    const cacheKey = `bundle_${bundleName}_${lang}`;
    if (this._cache.has(cacheKey)) {
      return JSON.parse(this._cache.get(cacheKey)!);
    }
    
    try {
      const response = await fetch(`/translations/${bundleName}/${lang}.json`);
      const bundle = await response.json();
      
      this._cache.set(cacheKey, JSON.stringify(bundle));
      return bundle;
    } catch (error) {
      console.error(`Failed to load translation bundle: ${bundleName}/${lang}`, error);
      return {};
    }
  }
  
  private escapeHtmlExcept(text: string, allowedTags: string[]): string {
    // 基本的なHTMLエスケープ
    let escaped = text
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;');
    
    // 許可されたタグをアンエスケープ
    allowedTags.forEach(tag => {
      const openTag = `&lt;${tag}&gt;`;
      const closeTag = `&lt;/${tag}&gt;`;
      escaped = escaped
        .replace(new RegExp(openTag, 'g'), `<${tag}>`)
        .replace(new RegExp(closeTag, 'g'), `</${tag}>`);
    });
    
    return escaped;
  }
  
  private getNestedMessage(obj: any, path: string): any {
    return path.split('.').reduce((current, key) => {
      return current && current[key] !== undefined ? current[key] : null;
    }, obj);
  }
}

export const advancedI18n = AdvancedI18n.getInstance();
```

### 2\. 国際化対応データテーブルコンポーネント

```svelte:src/lib/components/DataTable.svelte
<script lang="ts" generics="T extends Record<string, any>">
  import { onMount } from 'svelte';
  import * as m from '$lib/paraglide/messages';
  import { i18nUtils } from '$lib/i18n/utils';
  import { languageTag } from '$lib/paraglide/runtime';
  import { advancedI18n } from '$lib/i18n/advanced';
  
  interface Column<T> {
    key: keyof T;
    headerKey: string; // 翻訳キー
    sortable?: boolean;
    formatter?: (value: any, item: T, lang: string) => string;
    align?: 'left' | 'center' | 'right';
    width?: string;
  }
  
  export let data: T[] = [];
  export let columns: Column<T>[] = [];
  export let loading = false;
  export let emptyMessageKey = 'table_noData';
  export let itemsPerPage = 10;
  export let searchable = true;
  
  let sortColumn: keyof T | null = null;
  let sortDirection: 'asc' | 'desc' = 'asc';
  let searchQuery = '';
  let currentPage = 1;
  
  $: filteredData = searchQuery 
    ? data.filter(item => 
        Object.values(item).some(value => 
          String(value).toLowerCase().includes(searchQuery.toLowerCase())
        )
      )
    : data;
  
  $: sortedData = sortColumn 
    ? [...filteredData].sort((a, b) => {
        const aVal = a[sortColumn];
        const bVal = b[sortColumn];
        
        let comparison = 0;
        if (aVal > bVal) comparison = 1;
        if (aVal < bVal) comparison = -1;
        
        return sortDirection === 'desc' ? -comparison : comparison;
      })
    : filteredData;
  
  $: totalPages = Math.ceil(sortedData.length / itemsPerPage);
  $: startIndex = (currentPage - 1) * itemsPerPage;
  $: paginatedData = sortedData.slice(startIndex, startIndex + itemsPerPage);
  
  function handleSort(column: Column<T>) {
    if (!column.sortable) return;
    
    if (sortColumn === column.key) {
      sortDirection = sortDirection === 'asc' ? 'desc' : 'asc';
    } else {
      sortColumn = column.key;
      sortDirection = 'asc';
    }
  }
  
  function formatCellValue(value: any, item: T, column: Column<T>): string {
    if (column.formatter) {
      return column.formatter(value, item, languageTag());
    }
    
    // 基本的なフォーマット
    if (typeof value === 'number') {
      return i18nUtils.formatNumber(value);
    }
    
    if (value instanceof Date) {
      return i18nUtils.formatDate(value, { 
        year: 'numeric', 
        month: 'short', 
        day: 'numeric' 
      });
    }
    
    return String(value || '');
  }
  
  function changePage(page: number) {
    if (page >= 1 && page <= totalPages) {
      currentPage = page;
    }
  }
  
  $: paginationInfo = advancedI18n.plural(
    sortedData.length,
    {
      zero: m.table_pagination_zero(),
      one: m.table_pagination_one(),
      other: m.table_pagination_other({ count: sortedData.length })
    }
  );
</script>

<div class="data-table" dir={i18nUtils.isRTL() ? 'rtl' : 'ltr'}>
  {#if searchable}
    <div class="table-controls">
      <input 
        type="text"
        bind:value={searchQuery}
        placeholder={m.table_search_placeholder()}
        class="search-input"
      />
    </div>
  {/if}
  
  <div class="table-wrapper">
    <table class="table">
      <thead>
        <tr>
          {#each columns as column}
            <th 
              class="header-cell"
              class:sortable={column.sortable}
              class:sorted={sortColumn === column.key}
              class:asc={sortColumn === column.key && sortDirection === 'asc'}
              class:desc={sortColumn === column.key && sortDirection === 'desc'}
              style={column.width ? `width: ${column.width}` : ''}
              style:text-align={column.align || 'left'}
              on:click={() => handleSort(column)}
            >
              <span class="header-content">
                {advancedI18n.resolveDynamic(column.headerKey, {})}
                {#if column.sortable}
                  <svg class="sort-icon" viewBox="0 0 20 20" fill="currentColor">
                    <path d="M5 12l5-5 5 5H5z"/>
                  </svg>
                {/if}
              </span>
            </th>
          {/each}
        </tr>
      </thead>
      
      <tbody>
        {#if loading}
          <tr>
            <td colspan={columns.length} class="loading-cell">
              <div class="loading-spinner"></div>
              {m.table_loading()}
            </td>
          </tr>
        {:else if paginatedData.length === 0}
          <tr>
            <td colspan={columns.length} class="empty-cell">
              {advancedI18n.resolveDynamic(emptyMessageKey, {})}
            </td>
          </tr>
        {:else}
          {#each paginatedData as item, index}
            <tr class="data-row" class:even={index % 2 === 0}>
              {#each columns as column}
                <td 
                  class="data-cell"
                  style:text-align={column.align || 'left'}
                >
                  {formatCellValue(item[column.key], item, column)}
                </td>
              {/each}
            </tr>
          {/each}
        {/if}
      </tbody>
    </table>
  </div>
  
  {#if totalPages > 1}
    <div class="pagination">
      <div class="pagination-info">
        {paginationInfo}
      </div>
      
      <div class="pagination-controls">
        <button 
          on:click={() => changePage(currentPage - 1)}
          disabled={currentPage === 1}
          class="pagination-button"
        >
          {m.table_previous()}
        </button>
        
        {#each Array(totalPages) as _, i}
          {@const page = i + 1}
          {#if page === 1 || page === totalPages || Math.abs(page - currentPage) <= 2}
            <button
              on:click={() => changePage(page)}
              class="pagination-button"
              class:active={page === currentPage}
            >
              {page}
            </button>
          {:else if Math.abs(page - currentPage) === 3}
            <span class="pagination-ellipsis">...</span>
          {/if}
        {/each}
        
        <button 
          on:click={() => changePage(currentPage + 1)}
          disabled={currentPage === totalPages}
          class="pagination-button"
        >
          {m.table_next()}
        </button>
      </div>
    </div>
  {/if}
</div>

<style>
  .data-table {
    width: 100%;
    border: 1px solid #e5e7eb;
    border-radius: 0.5rem;
    overflow: hidden;
    background: white;
  }
  
  .table-controls {
    padding: 1rem;
    border-bottom: 1px solid #e5e7eb;
    background: #f9fafb;
  }
  
  .search-input {
    width: 100%;
    max-width: 300px;
    padding: 0.5rem;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem;
    font-size: 0.875rem;
  }
  
  .table-wrapper {
    overflow-x: auto;
  }
  
  .table {
    width: 100%;
    border-collapse: collapse;
  }
  
  .header-cell {
    padding: 0.75rem;
    background: #f9fafb;
    border-bottom: 1px solid #e5e7eb;
    font-weight: 600;
    color: #374151;
    white-space: nowrap;
  }
  
  .header-cell.sortable {
    cursor: pointer;
    user-select: none;
  }
  
  .header-cell.sortable:hover {
    background: #f3f4f6;
  }
  
  .header-content {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  
  .sort-icon {
    width: 1rem;
    height: 1rem;
    opacity: 0.5;
    transition: all 0.2s;
  }
  
  .header-cell.sorted .sort-icon {
    opacity: 1;
  }
  
  .header-cell.desc .sort-icon {
    transform: rotate(180deg);
  }
  
  .data-cell {
    padding: 0.75rem;
    border-bottom: 1px solid #f3f4f6;
  }
  
  .data-row.even {
    background: #fafafa;
  }
  
  .loading-cell, .empty-cell {
    padding: 2rem;
    text-align: center;
    color: #6b7280;
    font-style: italic;
  }
  
  .loading-cell {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 1rem;
  }
  
  .loading-spinner {
    width: 1.5rem;
    height: 1.5rem;
    border: 2px solid #e5e7eb;
    border-top: 2px solid #3b82f6;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  
  .pagination {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;
    border-top: 1px solid #e5e7eb;
    background: #f9fafb;
  }
  
  .pagination-info {
    font-size: 0.875rem;
    color: #6b7280;
  }
  
  .pagination-controls {
    display: flex;
    align-items: center;
    gap: 0.25rem;
  }
  
  .pagination-button {
    padding: 0.5rem 0.75rem;
    border: 1px solid #d1d5db;
    background: white;
    color: #374151;
    border-radius: 0.375rem;
    cursor: pointer;
    font-size: 0.875rem;
    transition: all 0.2s;
  }
  
  .pagination-button:hover:not(:disabled) {
    background: #f3f4f6;
  }
  
  .pagination-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  .pagination-button.active {
    background: #3b82f6;
    color: white;
    border-color: #3b82f6;
  }
  
  .pagination-ellipsis {
    padding: 0.5rem;
    color: #6b7280;
  }
  
  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }
</style>
```

## パフォーマンス最適化と本番対応

### 1\. ビルド最適化設定

```typescript:vite.config.js
import { paraglide } from '@inlang/paraglide-sveltekit/vite';
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [
    paraglide({
      project: './project.inlang',
      outdir: './src/lib/paraglide',
      // 未使用翻訳キーを削除
      treeShake: true,
      // 開発時のホットリロード
      watch: process.env.NODE_ENV === 'development'
    }),
    sveltekit()
  ],
  
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          // Paraglide関連を分離
          if (id.includes('paraglide')) {
            return 'paraglide';
          }
          // 各言語のメッセージを分離
          if (id.includes('/messages/')) {
            const lang = id.match(/\/messages\/(\w+)\./)?.[1];
            return lang ? `lang-${lang}` : 'messages';
          }
        }
      }
    }
  }
});
```

### 2\. SEOとメタデータの国際化

```svelte:src/app.html
<!DOCTYPE html>
<html lang="%paraglide.lang%" dir="%paraglide.dir%">
<head>
  <meta charset="utf-8" />
  <link rel="icon" href="%sveltekit.assets%/favicon.png" />
  <meta name="viewport" content="width=device-width" />
  %sveltekit.head%
</head>
<body data-sveltekit-preload-data="hover" class="%paraglide.lang%">
  <div style="display: contents">%sveltekit.body%</div>
</body>
</html>
```

```typescript:src/hooks.server.ts
import { ParaglideJS } from '@inlang/paraglide-sveltekit';
import { i18n } from '$lib/i18n';
import { sequence } from '@sveltejs/kit/hooks';

export const handle = sequence(
  ParaglideJS({
    i18n,
    // 言語検出ストラテジー
    strategy: {
      // URLパスから言語を検出
      path: {},
      // Accept-Languageヘッダーから検出
      acceptLanguage: {},
      // クッキーから検出
      cookie: {
        name: 'lang'
      }
    }
  }).handle(),
  
  // SEO用のhreflang追加
  async ({ event, resolve }) => {
    const response = await resolve(event);
    
    if (response.headers.get('content-type')?.includes('text/html')) {
      const lang = event.locals.paraglide?.lang || 'en';
      const canonical = new URL(event.url);
      
      // 現在のページの他言語版リンクを生成
      const alternateLinks = i18n.config.languageTags
        .map(l => {
          const url = new URL(canonical);
          url.pathname = url.pathname.replace(`/${lang}`, `/${l}`);
          return `<link rel="alternate" hreflang="${l}" href="${url.href}">`;
        })
        .join('\n');
      
      const html = await response.text();
      const modifiedHtml = html.replace('</head>', `${alternateLinks}\n</head>`);
      
      return new Response(modifiedHtml, {
        status: response.status,
        headers: response.headers
      });
    }
    
    return response;
  }
);
```

## まとめ

Paraglideは、従来のi18nライブラリの限界を打破する革新的なソリューションです。

**✅ Paraglideで実現できること**
 
> * 🚀 **ゼロランタイム**: コンパイル時処理による最高のパフォーマンス
> * 🔒 **完全な型安全性**: TypeScriptとの完璧な統合
> * 🌍 **SEO最適化**: 自動hreflang生成とメタデータ国際化
> * 📦 **Tree Shaking**: 未使用翻訳の自動削除
> * 🎯 **開発体験**: エラー検出とホットリロード対応

コンパイル時の型チェック、自動翻訳管理、パフォーマンス最適化により、世界中のユーザーに愛されるアプリケーションを効率よく開発できるようになりましたね！

次の国際化プロジェクトでは、ぜひParaglideの威力を活用して、ユーザーの母国語で心に響く素晴らしい体験を提供してください！