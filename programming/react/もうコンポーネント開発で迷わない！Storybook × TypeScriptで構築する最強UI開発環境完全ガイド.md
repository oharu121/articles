# もうコンポーネント開発で迷わない！Storybook × TypeScriptで構築する最強UI開発環境完全ガイド

## はじめに

Reactのコンポーネント開発で「デザインシステムの管理が大変」「コンポーネントのテストが面倒」「チームメンバーとの共有がうまくいかない」なんて悩みを抱えたことはありませんか？

新しいコンポーネントを作るたびに、アプリ全体を起動してブラウザで確認したり、propsの組み合わせを手動でテストしたり...効率が悪すぎますよね。しかも、デザイナーとの認識合わせやチームメンバーへのコンポーネント仕様共有も一苦労。

「もっとスマートに、効率よく、チーム全体でコンポーネント開発を進められる環境はないものか...」

この記事では、**Storybook × TypeScriptで実現する革新的なUI開発環境**の構築方法を実装例とともに徹底解説します！コンポーネントの分離開発、自動テスト、ドキュメント生成まで、現代のフロントエンド開発に欠かせない実践的な内容をお届けします。

## Storybookとは何か

Storybookは、**UIコンポーネントを分離された環境で開発・テスト・ドキュメント化できるツール**です。

**💡 Storybookの革新性**

> * コンポーネントをアプリから独立して開発可能
> * プロパティの組み合わせを視覚的にテスト
> * 自動ドキュメント生成とデザインシステム管理
> * アクセシビリティテストとビジュアルテスト統合

```typescript:Button.stories.ts
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Example/Button',
  component: Button,
  parameters: {
    layout: 'centered',
  },
  tags: ['autodocs'],
};

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: {
    primary: true,
    label: 'Button',
  },
};
```

## 基本的なセットアップ実装

### プロジェクトの初期化

```bash
# Reactプロジェクトの作成
npx create-react-app storybook-demo --template typescript
cd storybook-demo

# Storybookの導入
npx storybook@latest init
```

### TypeScript設定の最適化

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": [
      "dom",
      "dom.iterable",
      "ES6"
    ],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "baseUrl": "src",
    "paths": {
      "@/*": ["*"],
      "@/components/*": ["components/*"],
      "@/types/*": ["types/*"]
    }
  },
  "include": [
    "src",
    ".storybook"
  ]
}
```

### Storybook設定ファイル

```typescript:.storybook/main.ts
import type { StorybookConfig } from '@storybook/react-webpack5';
import path from 'path';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(js|jsx|mjs|ts|tsx)'],
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-onboarding',
    '@storybook/addon-interactions',
    '@storybook/addon-docs',
    '@storybook/addon-controls',
    '@storybook/addon-viewport',
    '@storybook/addon-backgrounds',
    '@storybook/addon-a11y',
    '@storybook/addon-measure',
    '@storybook/addon-outline'
  ],
  framework: {
    name: '@storybook/react-webpack5',
    options: {}
  },
  typescript: {
    reactDocgen: 'react-docgen-typescript',
    reactDocgenTypescriptOptions: {
      shouldExtractLiteralValuesFromEnum: true,
      propFilter: (prop) => (prop.parent ? !/node_modules/.test(prop.parent.fileName) : true),
    }
  },
  webpackFinal: async (config) => {
    // パスエイリアスの設定
    if (config.resolve) {
      config.resolve.alias = {
        ...config.resolve.alias,
        '@': path.resolve(__dirname, '../src'),
      };
    }
    return config;
  },
};

export default config;
```

```typescript:.storybook/preview.ts
import type { Preview } from '@storybook/react';
import '../src/index.css';

const preview: Preview = {
  parameters: {
    actions: { argTypesRegex: '^on[A-Z].*' },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      },
    },
    docs: {
      autodocs: 'tag',
    },
    backgrounds: {
      default: 'light',
      values: [
        {
          name: 'light',
          value: '#ffffff',
        },
        {
          name: 'dark',
          value: '#333333',
        },
        {
          name: 'gray',
          value: '#f5f5f5',
        },
      ],
    },
    viewport: {
      viewports: {
        mobile: {
          name: 'Mobile',
          styles: {
            width: '375px',
            height: '667px',
          },
        },
        tablet: {
          name: 'Tablet',
          styles: {
            width: '768px',
            height: '1024px',
          },
        },
        desktop: {
          name: 'Desktop',
          styles: {
            width: '1200px',
            height: '800px',
          },
        },
      },
    },
  },
};

export default preview;
```

## TypeScript型定義とコンポーネント設計

### 共通型定義の作成

```typescript:src/types/components.ts
import { ReactNode, MouseEvent, KeyboardEvent } from 'react';

export type ComponentSize = 'small' | 'medium' | 'large';
export type ComponentVariant = 'primary' | 'secondary' | 'success' | 'warning' | 'danger';
export type ComponentColor = 'blue' | 'green' | 'red' | 'yellow' | 'gray' | 'purple';

export interface BaseComponentProps {
  className?: string;
  testId?: string;
  children?: ReactNode;
}

export interface InteractiveComponentProps extends BaseComponentProps {
  disabled?: boolean;
  loading?: boolean;
  onClick?: (event: MouseEvent<HTMLElement>) => void;
  onKeyDown?: (event: KeyboardEvent<HTMLElement>) => void;
}

export interface FormFieldProps extends BaseComponentProps {
  id?: string;
  name?: string;
  label?: string;
  placeholder?: string;
  required?: boolean;
  error?: string;
  helpText?: string;
}

export interface LayoutProps extends BaseComponentProps {
  spacing?: number | string;
  padding?: number | string;
  margin?: number | string;
  align?: 'left' | 'center' | 'right';
  justify?: 'start' | 'center' | 'end' | 'between' | 'around';
}
```

### 高度なButtonコンポーネント実装

```typescript:src/components/Button/Button.tsx
import React, { forwardRef, ReactNode } from 'react';
import { ComponentSize, ComponentVariant, InteractiveComponentProps } from '@/types/components';
import './Button.css';

export interface ButtonProps extends InteractiveComponentProps {
  variant?: ComponentVariant;
  size?: ComponentSize;
  fullWidth?: boolean;
  leftIcon?: ReactNode;
  rightIcon?: ReactNode;
  type?: 'button' | 'submit' | 'reset';
  ariaLabel?: string;
  href?: string;
  target?: '_blank' | '_self' | '_parent' | '_top';
}

export const Button = forwardRef<HTMLButtonElement | HTMLAnchorElement, ButtonProps>(
  ({
    children,
    variant = 'primary',
    size = 'medium',
    disabled = false,
    loading = false,
    fullWidth = false,
    leftIcon,
    rightIcon,
    className = '',
    testId = 'button',
    type = 'button',
    ariaLabel,
    href,
    target,
    onClick,
    onKeyDown,
    ...rest
  }, ref) => {
    const classes = [
      'btn',
      `btn--${variant}`,
      `btn--${size}`,
      fullWidth && 'btn--full-width',
      (disabled || loading) && 'btn--disabled',
      loading && 'btn--loading',
      className
    ].filter(Boolean).join(' ');

    const content = (
      <>
        {leftIcon && <span className="btn__icon btn__icon--left">{leftIcon}</span>}
        {loading && <div className="btn__spinner" />}
        <span className={`btn__text ${loading ? 'btn__text--loading' : ''}`}>
          {children}
        </span>
        {rightIcon && <span className="btn__icon btn__icon--right">{rightIcon}</span>}
      </>
    );

    const commonProps = {
      className: classes,
      'data-testid': testId,
      'aria-label': ariaLabel,
      'aria-disabled': disabled || loading,
      onKeyDown,
      ...rest
    };

    if (href) {
      return (
        <a
          ref={ref as React.Ref<HTMLAnchorElement>}
          href={disabled ? undefined : href}
          target={target}
          onClick={disabled || loading ? undefined : onClick}
          {...commonProps}
        >
          {content}
        </a>
      );
    }

    return (
      <button
        ref={ref as React.Ref<HTMLButtonElement>}
        type={type}
        disabled={disabled || loading}
        onClick={onClick}
        {...commonProps}
      >
        {content}
      </button>
    );
  }
);

Button.displayName = 'Button';
```

```css:src/components/Button/Button.css
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  border: none;
  border-radius: 0.375rem;
  font-family: inherit;
  font-weight: 500;
  text-align: center;
  text-decoration: none;
  white-space: nowrap;
  cursor: pointer;
  transition: all 0.2s ease-in-out;
  position: relative;
  overflow: hidden;
}

/* サイズバリエーション */
.btn--small {
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  line-height: 1.25rem;
}

.btn--medium {
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  line-height: 1.5rem;
}

.btn--large {
  padding: 1rem 2rem;
  font-size: 1.125rem;
  line-height: 1.75rem;
}

/* バリアントスタイル */
.btn--primary {
  background-color: #3b82f6;
  color: white;
}

.btn--primary:hover:not(.btn--disabled) {
  background-color: #2563eb;
}

.btn--secondary {
  background-color: transparent;
  color: #3b82f6;
  border: 1px solid #3b82f6;
}

.btn--secondary:hover:not(.btn--disabled) {
  background-color: #3b82f6;
  color: white;
}

.btn--success {
  background-color: #10b981;
  color: white;
}

.btn--success:hover:not(.btn--disabled) {
  background-color: #059669;
}

.btn--warning {
  background-color: #f59e0b;
  color: white;
}

.btn--warning:hover:not(.btn--disabled) {
  background-color: #d97706;
}

.btn--danger {
  background-color: #ef4444;
  color: white;
}

.btn--danger:hover:not(.btn--disabled) {
  background-color: #dc2626;
}

/* 状態スタイル */
.btn--disabled {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
}

.btn--loading {
  cursor: wait;
}

.btn--full-width {
  width: 100%;
}

/* アイコンとスピナー */
.btn__icon {
  display: flex;
  align-items: center;
  justify-content: center;
}

.btn__spinner {
  width: 1rem;
  height: 1rem;
  border: 2px solid transparent;
  border-top: 2px solid currentColor;
  border-radius: 50%;
  animation: btn-spin 1s linear infinite;
  position: absolute;
}

.btn__text--loading {
  opacity: 0;
}

@keyframes btn-spin {
  to {
    transform: rotate(360deg);
  }
}

/* フォーカススタイル */
.btn:focus-visible {
  outline: 2px solid #3b82f6;
  outline-offset: 2px;
}

/* ホバー効果 */
.btn:not(.btn--disabled):active {
  transform: translateY(1px);
}
```

## Storybookストーリーの実装

### 基本的なButtonストーリー

```typescript:src/components/Button/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { action } from '@storybook/addon-actions';
import { Button, ButtonProps } from './Button';
import { HeartIcon, ArrowRightIcon } from '@heroicons/react/24/outline';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  parameters: {
    layout: 'centered',
    docs: {
      description: {
        component: `
汎用的なButtonコンポーネントです。さまざまなバリエーション、サイズ、状態をサポートしています。

## 使用方法

\`\`\`tsx
import { Button } from '@/components/Button';

<Button variant="primary" size="medium" onClick={handleClick}>
  クリック
</Button>
\`\`\`

## 特徴

- 5つのバリアント（primary, secondary, success, warning, danger）
- 3つのサイズ（small, medium, large）
- ローディング状態とアイコン対応
- アクセシビリティ完全対応
- リンクとしても使用可能
        `
      }
    }
  },
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: { type: 'select' },
      options: ['primary', 'secondary', 'success', 'warning', 'danger'],
      description: 'ボタンの見た目のバリエーション',
    },
    size: {
      control: { type: 'select' },
      options: ['small', 'medium', 'large'],
      description: 'ボタンのサイズ',
    },
    disabled: {
      control: { type: 'boolean' },
      description: 'ボタンを無効化するかどうか',
    },
    loading: {
      control: { type: 'boolean' },
      description: 'ローディング状態を表示するかどうか',
    },
    fullWidth: {
      control: { type: 'boolean' },
      description: '親要素の幅いっぱいに表示するかどうか',
    },
    children: {
      control: { type: 'text' },
      description: 'ボタン内に表示するテキスト',
    },
    onClick: {
      action: 'clicked',
      description: 'クリック時のイベントハンドラー',
    },
  },
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

// 基本的なストーリー
export const Default: Story = {
  args: {
    children: 'Button',
    onClick: action('button-click'),
  },
};

export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'Primary Button',
    onClick: action('primary-click'),
  },
};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
    children: 'Secondary Button',
    onClick: action('secondary-click'),
  },
};

// サイズバリエーション
export const Sizes: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', alignItems: 'center' }}>
      <Button size="small" onClick={action('small-click')}>Small</Button>
      <Button size="medium" onClick={action('medium-click')}>Medium</Button>
      <Button size="large" onClick={action('large-click')}>Large</Button>
    </div>
  ),
  parameters: {
    docs: {
      description: {
        story: '3つの異なるサイズのボタンを表示します。',
      },
    },
  },
};

// 全バリエーション
export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fit, minmax(150px, 1fr))', gap: '1rem' }}>
      <Button variant="primary" onClick={action('primary-click')}>Primary</Button>
      <Button variant="secondary" onClick={action('secondary-click')}>Secondary</Button>
      <Button variant="success" onClick={action('success-click')}>Success</Button>
      <Button variant="warning" onClick={action('warning-click')}>Warning</Button>
      <Button variant="danger" onClick={action('danger-click')}>Danger</Button>
    </div>
  ),
  parameters: {
    docs: {
      description: {
        story: '利用可能なすべてのバリアントを表示します。',
      },
    },
  },
};

// 状態バリエーション
export const States: Story = {
  render: () => (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '1rem', alignItems: 'flex-start' }}>
      <Button onClick={action('normal-click')}>Normal</Button>
      <Button disabled>Disabled</Button>
      <Button loading>Loading</Button>
      <Button fullWidth>Full Width</Button>
    </div>
  ),
  parameters: {
    docs: {
      description: {
        story: 'ボタンの様々な状態を表示します。',
      },
    },
  },
};

// アイコン付きボタン
export const WithIcons: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', flexWrap: 'wrap' }}>
      <Button 
        leftIcon={<HeartIcon style={{ width: '1rem', height: '1rem' }} />}
        onClick={action('heart-click')}
      >
        Like
      </Button>
      <Button 
        rightIcon={<ArrowRightIcon style={{ width: '1rem', height: '1rem' }} />}
        onClick={action('arrow-click')}
      >
        Next
      </Button>
      <Button 
        leftIcon={<HeartIcon style={{ width: '1rem', height: '1rem' }} />}
        rightIcon={<ArrowRightIcon style={{ width: '1rem', height: '1rem' }} />}
        onClick={action('both-icons-click')}
      >
        Both Icons
      </Button>
    </div>
  ),
  parameters: {
    docs: {
      description: {
        story: 'アイコンが付いたボタンの例です。',
      },
    },
  },
};

// リンクとして使用
export const AsLink: Story = {
  args: {
    href: 'https://example.com',
    target: '_blank',
    children: 'External Link',
    rightIcon: <ArrowRightIcon style={{ width: '1rem', height: '1rem' }} />,
  },
  parameters: {
    docs: {
      description: {
        story: 'ボタンをリンクとして使用する例です。',
      },
    },
  },
};

// インタラクティブなプレイグラウンド
export const Playground: Story = {
  args: {
    variant: 'primary',
    size: 'medium',
    children: 'Playground Button',
    disabled: false,
    loading: false,
    fullWidth: false,
    onClick: action('playground-click'),
  },
  parameters: {
    docs: {
      description: {
        story: 'すべてのプロパティを自由に調整できるプレイグラウンドです。',
      },
    },
  },
};
```

## TypeScriptでの高度な活用例

### フォームコンポーネントの実装

```typescript:src/components/Form/FormField.tsx
import React, { ReactNode, useId } from 'react';
import { FormFieldProps } from '@/types/components';
import './FormField.css';

export interface FormFieldWrapperProps extends FormFieldProps {
  children: ReactNode;
  labelPosition?: 'top' | 'left' | 'floating';
  required?: boolean;
  optional?: boolean;
}

export const FormField: React.FC<FormFieldWrapperProps> = ({
  children,
  label,
  error,
  helpText,
  required = false,
  optional = false,
  labelPosition = 'top',
  className = '',
  testId = 'form-field'
}) => {
  const fieldId = useId();
  const errorId = error ? `${fieldId}-error` : undefined;
  const helpId = helpText ? `${fieldId}-help` : undefined;

  const fieldClasses = [
    'form-field',
    `form-field--${labelPosition}`,
    error && 'form-field--error',
    className
  ].filter(Boolean).join(' ');

  const renderLabel = () => {
    if (!label) return null;

    return (
      <label htmlFor={fieldId} className="form-field__label">
        {label}
        {required && <span className="form-field__required" aria-label="必須">*</span>}
        {optional && <span className="form-field__optional">(任意)</span>}
      </label>
    );
  };

  return (
    <div className={fieldClasses} data-testid={testId}>
      {labelPosition !== 'floating' && renderLabel()}
      
      <div className="form-field__input-wrapper">
        {React.cloneElement(children as React.ReactElement, {
          id: fieldId,
          'aria-describedby': [errorId, helpId].filter(Boolean).join(' ') || undefined,
          'aria-invalid': !!error,
          'aria-required': required,
        })}
        {labelPosition === 'floating' && renderLabel()}
      </div>

      {helpText && (
        <p id={helpId} className="form-field__help">
          {helpText}
        </p>
      )}

      {error && (
        <p id={errorId} className="form-field__error" role="alert">
          {error}
        </p>
      )}
    </div>
  );
};
```

### 高度なInputコンポーネント

```typescript:src/components/Input/Input.tsx
import React, { forwardRef, useState, useCallback, ReactNode } from 'react';
import { ComponentSize, FormFieldProps } from '@/types/components';
import { EyeIcon, EyeSlashIcon } from '@heroicons/react/24/outline';
import './Input.css';

export interface InputProps extends Omit<FormFieldProps, 'children'> {
  type?: 'text' | 'email' | 'password' | 'number' | 'tel' | 'url' | 'search';
  size?: ComponentSize;
  value?: string;
  defaultValue?: string;
  maxLength?: number;
  minLength?: number;
  pattern?: string;
  autoComplete?: string;
  autoFocus?: boolean;
  readOnly?: boolean;
  leftIcon?: ReactNode;
  rightIcon?: ReactNode;
  onClear?: () => void;
  onChange?: (value: string, event: React.ChangeEvent<HTMLInputElement>) => void;
  onFocus?: (event: React.FocusEvent<HTMLInputElement>) => void;
  onBlur?: (event: React.FocusEvent<HTMLInputElement>) => void;
  onKeyDown?: (event: React.KeyboardEvent<HTMLInputElement>) => void;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(({
  type = 'text',
  size = 'medium',
  value,
  defaultValue,
  placeholder,
  disabled = false,
  readOnly = false,
  required = false,
  maxLength,
  minLength,
  pattern,
  autoComplete,
  autoFocus = false,
  leftIcon,
  rightIcon,
  error,
  className = '',
  testId = 'input',
  onClear,
  onChange,
  onFocus,
  onBlur,
  onKeyDown,
  ...rest
}, ref) => {
  const [showPassword, setShowPassword] = useState(false);
  const [isFocused, setIsFocused] = useState(false);

  const inputType = type === 'password' && showPassword ? 'text' : type;
  const hasValue = value !== undefined ? value.length > 0 : false;
  const showClearButton = hasValue && onClear && !disabled && !readOnly;
  const showPasswordToggle = type === 'password' && !disabled && !readOnly;

  const inputClasses = [
    'input',
    `input--${size}`,
    error && 'input--error',
    disabled && 'input--disabled',
    readOnly && 'input--readonly',
    isFocused && 'input--focused',
    leftIcon && 'input--with-left-icon',
    (rightIcon || showClearButton || showPasswordToggle) && 'input--with-right-icon',
    className
  ].filter(Boolean).join(' ');

  const handleChange = useCallback((event: React.ChangeEvent<HTMLInputElement>) => {
    onChange?.(event.target.value, event);
  }, [onChange]);

  const handleFocus = useCallback((event: React.FocusEvent<HTMLInputElement>) => {
    setIsFocused(true);
    onFocus?.(event);
  }, [onFocus]);

  const handleBlur = useCallback((event: React.FocusEvent<HTMLInputElement>) => {
    setIsFocused(false);
    onBlur?.(event);
  }, [onBlur]);

  const togglePasswordVisibility = useCallback(() => {
    setShowPassword(prev => !prev);
  }, []);

  return (
    <div className="input__wrapper">
      {leftIcon && (
        <div className="input__icon input__icon--left">
          {leftIcon}
        </div>
      )}

      <input
        ref={ref}
        type={inputType}
        value={value}
        defaultValue={defaultValue}
        placeholder={placeholder}
        disabled={disabled}
        readOnly={readOnly}
        required={required}
        maxLength={maxLength}
        minLength={minLength}
        pattern={pattern}
        autoComplete={autoComplete}
        autoFocus={autoFocus}
        className={inputClasses}
        data-testid={testId}
        onChange={handleChange}
        onFocus={handleFocus}
        onBlur={handleBlur}
        onKeyDown={onKeyDown}
        {...rest}
      />

      <div className="input__right-icons">
        {showClearButton && (
          <button
            type="button"
            className="input__clear-button"
            onClick={onClear}
            aria-label="入力をクリア"
          >
            ×
          </button>
        )}

        {showPasswordToggle && (
          <button
            type="button"
            className="input__password-toggle"
            onClick={togglePasswordVisibility}
            aria-label={showPassword ? 'パスワードを隠す' : 'パスワードを表示'}
          >
            {showPassword ? (
              <EyeSlashIcon className="input__icon-svg" />
            ) : (
              <EyeIcon className="input__icon-svg" />
            )}
          </button>
        )}

        {rightIcon && (
          <div className="input__icon input__icon--right">
            {rightIcon}
          </div>
        )}
      </div>
    </div>
  );
});

Input.displayName = 'Input';
```

### 包括的なInputストーリー

```typescript:src/components/Input/Input.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { action } from '@storybook/addon-actions';
import { useState } from 'react';
import { Input } from './Input';
import { FormField } from '../Form/FormField';
import { 
  MagnifyingGlassIcon, 
  EnvelopeIcon, 
  UserIcon,
  PhoneIcon 
} from '@heroicons/react/24/outline';

const meta: Meta<typeof Input> = {
  title: 'Components/Input',
  component: Input,
  parameters: {
    layout: 'centered',
    docs: {
      description: {
        component: `
高機能なInputコンポーネントです。様々な入力タイプ、バリデーション、アイコン表示に対応しています。

## 主な機能

- 複数の入力タイプサポート（text, email, password, number, etc.）
- アイコン表示（左右両方対応）
- パスワード表示/非表示切り替え
- クリアボタン
- アクセシビリティ完全対応
- フォームバリデーション統合

## 使用方法

\`\`\`tsx
import { Input } from '@/components/Input';
import { FormField } from '@/components/Form/FormField';

<FormField label="メールアドレス" required>
  <Input 
    type="email" 
    placeholder="example@domain.com"
    onChange={(value) => console.log(value)}
  />
</FormField>
\`\`\`
        `
      }
    }
  },
  tags: ['autodocs'],
  argTypes: {
    type: {
      control: { type: 'select' },
      options: ['text', 'email', 'password', 'number', 'tel', 'url', 'search'],
      description: '入力フィールドのタイプ',
    },
    size: {
      control: { type: 'select' },
      options: ['small', 'medium', 'large'],
      description: '入力フィールドのサイズ',
    },
    disabled: {
      control: { type: 'boolean' },
      description: '入力を無効化するかどうか',
    },
    readOnly: {
      control: { type: 'boolean' },
      description: '読み取り専用にするかどうか',
    },
    required: {
      control: { type: 'boolean' },
      description: '必須フィールドかどうか',
    },
    error: {
      control: { type: 'text' },
      description: 'エラーメッセージ',
    },
    placeholder: {
      control: { type: 'text' },
      description: 'プレースホルダーテキスト',
    },
    onChange: {
      action: 'changed',
      description: '値変更時のイベントハンドラー',
    },
  },
};

export default meta;
type Story = StoryObj<typeof meta>;

// 基本的なストーリー
export const Default: Story = {
  args: {
    placeholder: '何かを入力してください',
    onChange: action('input-change'),
  },
};

// サイズバリエーション
export const Sizes: Story = {
  render: () => (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '1rem', width: '300px' }}>
      <Input size="small" placeholder="Small input" onChange={action('small-change')} />
      <Input size="medium" placeholder="Medium input" onChange={action('medium-change')} />
      <Input size="large" placeholder="Large input" onChange={action('large-change')} />
    </div>
  ),
};

// 入力タイプ
export const InputTypes: Story = {
  render: () => (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '1rem', width: '300px' }}>
      <FormField label="テキスト">
        <Input type="text" placeholder="テキストを入力" onChange={action('text-change')} />
      </FormField>
      <FormField label="メールアドレス">
        <Input type="email" placeholder="email@example.com" onChange={action('email-change')} />
      </FormField>
      <FormField label="パスワード">
        <Input type="password" placeholder="パスワードを入力" onChange={action('password-change')} />
      </FormField>
      <FormField label="数値">
        <Input type="number" placeholder="123" onChange={action('number-change')} />
      </FormField>
      <FormField label="電話番号">
        <Input type="tel" placeholder="090-1234-5678" onChange={action('tel-change')} />
      </FormField>
    </div>
  ),
};

// アイコン付き入力
export const WithIcons: Story = {
  render: () => (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '1rem', width: '300px' }}>
      <FormField label="検索">
        <Input 
          type="search"
          placeholder="検索キーワード"
          leftIcon={<MagnifyingGlassIcon style={{ width: '1rem', height: '1rem' }} />}
          onChange={action('search-change')}
        />
      </FormField>
      <FormField label="メールアドレス">
        <Input 
          type="email"
          placeholder="email@example.com"
          leftIcon={<EnvelopeIcon style={{ width: '1rem', height: '1rem' }} />}
          onChange={action('email-icon-change')}
        />
      </FormField>
      <FormField label="ユーザー名">
        <Input 
          type="text"
          placeholder="ユーザー名"
          leftIcon={<UserIcon style={{ width: '1rem', height: '1rem' }} />}
          onChange={action('username-change')}
        />
      </FormField>
    </div>
  ),
};

// 状態バリエーション
export const States: Story = {
  render: () => (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '1rem', width: '300px' }}>
      <FormField label="通常状態">
        <Input placeholder="通常の入力フィールド" onChange={action('normal-change')} />
      </FormField>
      <FormField label="無効状態">
        <Input disabled placeholder="無効化された入力フィールド" />
      </FormField>
      <FormField label="読み取り専用">
        <Input readOnly value="読み取り専用の値" />
      </FormField>
      <FormField label="エラー状態" error="このフィールドは必須です">
        <Input error="このフィールドは必須です" placeholder="エラーのある入力フィールド" />
      </FormField>
    </div>
  ),
};

// クリア機能付き
export const WithClearButton: Story = {
  render: () => {
    const [value, setValue] = useState('クリアできるテキスト');
    
    return (
      <div style={{ width: '300px' }}>
        <FormField label="クリア機能付き入力">
          <Input
            value={value}
            placeholder="何かを入力してください"
            onClear={() => setValue('')}
            onChange={(newValue) => setValue(newValue)}
          />
        </FormField>
      </div>
    );
  },
};

// バリデーション例
export const WithValidation: Story = {
  render: () => {
    const [email, setEmail] = useState('');
    const [emailError, setEmailError] = useState('');

    const validateEmail = (value: string) => {
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!value) {
        setEmailError('メールアドレスは必須です');
      } else if (!emailRegex.test(value)) {
        setEmailError('有効なメールアドレスを入力してください');
      } else {
        setEmailError('');
      }
    };

    return (
      <div style={{ width: '300px' }}>
        <FormField 
          label="メールアドレス" 
          required 
          error={emailError}
          helpText="example@domain.com の形式で入力してください"
        >
          <Input
            type="email"
            value={email}
            placeholder="email@example.com"
            leftIcon={<EnvelopeIcon style={{ width: '1rem', height: '1rem' }} />}
            error={emailError}
            onChange={(value) => {
              setEmail(value);
              validateEmail(value);
            }}
          />
        </FormField>
      </div>
    );
  },
};
```

## 自動テストとアクセシビリティ

### Testing Libraryとの統合

```typescript:src/components/Button/Button.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';
import { HeartIcon } from '@heroicons/react/24/outline';

describe('Button', () => {
  it('正常にレンダリングされる', () => {
    render(<Button>Test Button</Button>);
    expect(screen.getByRole('button', { name: 'Test Button' })).toBeInTheDocument();
  });

  it('クリックイベントが正常に動作する', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Clickable</Button>);
    
    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('無効化状態では클릭イベントが発火しない', async () => {
    const handleClick = jest.fn();
    render(<Button disabled onClick={handleClick}>Disabled</Button>);
    
    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).not.toHaveBeenCalled();
  });

  it('ローディング状態では클릭イベントが発火しない', async () => {
    const handleClick = jest.fn();
    render(<Button loading onClick={handleClick}>Loading</Button>);
    
    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).not.toHaveBeenCalled();
  });

  it('適切なaria属性が設定される', () => {
    render(<Button disabled ariaLabel="Close dialog">×</Button>);
    
    const button = screen.getByRole('button');
    expect(button).toHaveAttribute('aria-disabled', 'true');
    expect(button).toHaveAttribute('aria-label', 'Close dialog');
  });

  it('リンクとして正常に動作する', () => {
    render(<Button href="https://example.com" target="_blank">Link</Button>);
    
    const link = screen.getByRole('link');
    expect(link).toHaveAttribute('href', 'https://example.com');
    expect(link).toHaveAttribute('target', '_blank');
  });

  it('キーボードナビゲーションが正常に動作する', async () => {
    const handleKeyDown = jest.fn();
    render(<Button onKeyDown={handleKeyDown}>Keyboard</Button>);
    
    const button = screen.getByRole('button');
    await userEvent.type(button, '{enter}');
    
    expect(handleKeyDown).toHaveBeenCalled();
  });
});
```

### Storybookアドオンの活用

```typescript:.storybook/main.ts
const config: StorybookConfig = {
  // ... 既存の設定

  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-a11y',           // アクセシビリティテスト
    '@storybook/addon-viewport',       // レスポンシブテスト
    '@storybook/addon-backgrounds',    // 背景色テスト
    '@storybook/addon-interactions',   // インタラクションテスト
    '@storybook/addon-coverage',       // コードカバレッジ
    '@storybook/addon-design-tokens',  // デザイントークン
    'storybook-addon-designs',         // Figmaデザイン統合
  ],
};
```

## パフォーマンス最適化と本番対応

### ビルド最適化設定

```typescript:vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Storybook関連を分離
          'storybook': ['@storybook/react'],
          // UI컴포넌트を分離
          'ui-components': [
            './src/components/Button',
            './src/components/Input',
            './src/components/Form'
          ],
        },
      },
    },
  },
});
```

### CI/CDでのStorybook統合

```yaml:.github/workflows/storybook.yml
name: Storybook Tests
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run Storybook tests
        run: |
          npm run build-storybook
          npm run test-storybook
      
      - name: Visual regression tests
        run: npm run chromatic
        env:
          CHROMATIC_PROJECT_TOKEN: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
      
      - name: Accessibility tests
        run: npm run storybook:a11y
```

## まとめ

Storybookは、現代のコンポーネント開発において欠かせない強力なツールです。

> ✅ **Storybook × TypeScriptで実現できること**
> 
> * 🧩 **分離開発**: コンポーネントをアプリから独立してテスト
> * 📚 **自動ドキュメント**: PropTypesから自動生成される文書
> * 🎨 **デザインシステム**: 一貫したUI管理とスタイルガイド
> * ♿ **アクセシビリティ**: 自動a11yテストとベストプラクティス
> * 🔄 **CI/CD統合**: 自動テストとビジュアルリグレッションテスト

TypeScriptとの完璧な統合により、型安全性を保ちながら効率的なコンポーネント開発が可能になりましたね！

次のUIプロジェクトでは、ぜひStorybookを活用して、チーム全体で共有できる高品質なデザインシステムを構築してください！

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"id": "1", "content": "Storybook\u306e\u9069\u5207\u306a\u914d\u7f6e\u30d5\u30a9\u30eb\u30c0\u3092\u6c7a\u5b9a\u3059\u308b", "status": "completed"}, {"id": "2", "content": "\u8a18\u4e8b\u30bf\u30a4\u30c8\u30eb\u3068\u69cb\u6210\u3092\u4f01\u753b\u3059\u308b", "status": "completed"}, {"id": "3", "content": "Storybook\u306e\u57fa\u672c\u7684\u306a\u5b9f\u88c5\u4f8b\u3092\u4f5c\u6210\u3059\u308b", "status": "completed"}, {"id": "4", "content": "TypeScript\u3067\u306e\u9ad8\u5ea6\u306a\u6d3b\u7528\u4f8b\u3092\u5b9f\u88c5\u3059\u308b", "status": "completed"}, {"id": "5", "content": "\u8a18\u4e8b\u3092\u57f7\u7b46\u30fb\u5b8c\u6210\u3055\u305b\u308b", "status": "completed"}]