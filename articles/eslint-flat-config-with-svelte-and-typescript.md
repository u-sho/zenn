---
title: "Flat Config で Svelte＋TypeScript を ESLint する"
emoji: "💄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "eslint", "typescript"]
published: true
published_at: 2024-04-29 21:00
---

ESLintでは設定ファイルを[複雑な旧形式 eslintrc](https://eslint.org/docs/v8.x/use/configure/configuration-files) から改め、[独自ルールの少ない新形式 flat config](https://eslint.org/blog/2022/08/new-config-system-part-2/#the-goals-of-flat-config) に移行しています。
[先日リリースされた ESLint v9](https://eslint.org/blog/2024/04/eslint-v9.0.0-released/)では flat config が旧形式に代わってデフォルトの設定ファイルとなっており、次のメジャーバージョンであるESLint v10以降は旧形式の設定ファイルはサポートされなくなります。(_cf._ [Flat config rollout plans - ESLint](https://eslint.org/blog/2023/10/flat-config-rollout-plans/))
ESLint v9がリリースされた今、趣味で作っている [SvelteKit](https://kit.svelte.jp/) プロジェクトを flat config 移行することにしました。

忙しい人は [§3 移行まとめ](#§3-移行まとめ) を読むと雰囲気がつかめるかもしれません。
Svelte＋TypeScript の構成に限らず、flat config 移行でのつまづきをなくすように書いたつもりです。

## §0　前提等

[Svelte](https://svelte.jp/) ないし [SvelteKit](https://kit.svelte.jp/) のプロジェクトでは、リンタとして ESLint、フォーマッタとして Prettier を利用する構成が一般的かと思います。
この場合、ESLint プラグインとしては、主として [typescript-eslint](https://typescript-eslint.io/), [eslint-plugin-svelte](https://sveltejs.github.io/eslint-plugin-svelte/), [eslint-config-prettier](https://github.com/prettier/eslint-config-prettier#readme) を使用されているはずです。
本記事では、これら3プラグインを flat config を介して設定する方法を紹介します。

### §0.1　各種ライブラリのバージョン

Flat config への移行を行う前に各種ライブラリのバージョンアップをしておきます。
予めアンインストールして古い依存関係を残さないことで依存するパッケージを最新に保ちメンテナンスを楽にする効果が期待できます（npm の場合、[`lockfileVersion`](https://docs.npmjs.com/cli/v10/configuring-npm/package-lock-json#lockfileversion)が`3`なら特に問題ないかもしれません）。

```sh
npm un   eslint typescript-eslint eslint-plugin-svelte eslint-config-prettier
npm i -D eslint typescript-eslint eslint-plugin-svelte eslint-config-prettier
```

なお、本記事執筆時点 (2024-04-29) では typescript-eslint の [ESLint v9 への移行対応](https://github.com/typescript-eslint/typescript-eslint/issues/8211) が終了していないことなどから、以下のバージョンを使用しています。

| npm パッケージ名 | バージョン | 備考 |
| ------------- | -------- | ---- |
| [eslint](https://www.npmjs.com/package/eslint) | `8.57.0` | 最新は v9.1.0 ですが、`typescript-eslint`パッケージが対応していないため`^8.0.0`を用います |
| [typescript-eslint](https://www.npmjs.com/package/typescript-eslint) | `7.7.1` | v7から`@typescript-eslint/parser`と`@typescript-eslint/eslint-plugin`が統合されました。ESLint`^8.56.0`に対応しています |
| [eslint-plugin-svelte](https://www.npmjs.com/package/eslint-plugin-svelte) | `2.38.0` | Svelte`^3.37.0 \|\| ^4.0.0 \|\| ^5.0.0-next.112`に対応しています |
| [eslint-config-prettier](https://www.npmjs.com/package/eslint-config-prettier) | `9.1.0` | ESLint`>=7.0.0`に対応しています |
| [@types/node](https://www.npmjs.com/package/@types/node) | `20.12.7` | Node.js`>=20.11.0`で使える`import.meta.dirname`の型情報を使います |

また、[`import.meta.dirname`](https://nodejs.org/api/esm.html#importmetadirname)を使いたいので Node.js`^20.11.0`を使います。
> CommonJS module やこれより古い Node.js をお使いの場合は、[代わりに`__dirname`をお使いください](https://stackoverflow.com/questions/46745014/alternative-for-dirname-in-node-js-when-using-es6-modules)。
> [Linting with Type Information | typescript-eslint](https://typescript-eslint.io/getting-started/typed-linting) より引用・和訳

## §1　ESLint 公式のマイグレーションガイドにしたがって Flat Config に移行する

趣味で作っている [SvelteKit](https://kit.svelte.jp/) プロジェクトの`.eslintrc.cjs`ファイルを [ESLint 公式のマイグレーションガイド](https://eslint.org/docs/v8.x/use/configure/migration-guide)にしたがって flat config に移行してみます。

### §1.0　移行前のファイルの確認

旧設定方式では一つの大きな設定用オブジェクトで全体の設定を行い、さらにその中の`overrides`プロパティで`files`ごとの設定を行っていましたが、flat config では`files`ごとに設定用オブジェクトを分割し、それらを配列に順に格納することで設定を行います。
つまり、flat config は旧設定方式における`overrides`プロパティのみで設定するイメージです。

:::details 移行前の設定ファイルはこんな感じです。
`.eslintignore`ファイルを用いず、`ignorePatterns`で除外ファイルを設定しています。

```js:.eslintrc.cjs
// eslint-disable-next-line @typescript-eslint/no-unsafe-member-access
const isProduction = () => process.env.NODE_ENV === 'production';

/** @type {import('eslint').Linter.Config} */
module.exports = {
	root: true,
	reportUnusedDisableDirectives: true,
	ignorePatterns: [
		'.svelte-kit/',
		'.vercel/', // adapter-vercel output dir
		'.vercel_build_output/', // old output dir
		'static/',
		'build/',
		'coverage/', // vitest coverage
		'vitest.config.ts.timestamp*', // vite temp files
		'node_modules/'
	],
	plugins: ['@typescript-eslint'],
	extends: [
		'eslint:recommended',
		'plugin:@typescript-eslint/strict-type-checked',
		'plugin:@typescript-eslint/stylistic-type-checked',
		'prettier',
		'plugin:svelte/all',
		'plugin:svelte/prettier'
	],
	parser: '@typescript-eslint/parser',
	parserOptions: {
		sourceType: 'module',
		ecmaVersion: 'latest',
		project: './tsconfig.eslint.json',
		extraFileExtensions: ['.svelte']
	},
	env: {
		browser: true,
		es2022: true,
		node: true
	},
	rules: {
		'no-console': isProduction() ? 'error' : 'off',

		eqeqeq: ['error', 'always', { null: 'ignore' }],
		'no-duplicate-imports': ['error', { includeExports: true }],
		'no-restricted-imports': [
			'error',
			{ patterns: [{ group: ['../*', 'src/lib/*'], message: 'use `$lib/*` instead' }] }
		],
		'no-trailing-spaces': 'warn',
		'no-unused-expressions': 'error',
		'no-var': 'error',
		'prefer-const': 'error',

		'svelte/no-reactive-reassign': ['error', { props: true }],
		'svelte/block-lang': ['error', { script: 'ts', style: null }],
		'svelte/no-inline-styles': 'off',
		'svelte/no-unused-class-name': 'warn',
		'svelte/no-useless-mustaches': 'warn',
		'svelte/no-restricted-html-elements': 'off',
		'svelte/require-optimized-style-attribute': 'warn',
		'svelte/sort-attributes': 'off',
		'svelte/experimental-require-slot-types': 'off',
		'svelte/experimental-require-strict-events': 'off',

		'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
		'@typescript-eslint/consistent-type-definitions': ['error', 'type'],
		'@typescript-eslint/consistent-type-exports': 'error',
		'@typescript-eslint/consistent-type-imports': 'error',
		'@typescript-eslint/explicit-function-return-type': 'error',
		'@typescript-eslint/explicit-member-accessibility': ['warn', { accessibility: 'no-public' }],
		'@typescript-eslint/member-delimiter-style': 'warn',
		'@typescript-eslint/method-signature-style': 'error',
		camelcase: 'off',
		'@typescript-eslint/naming-convention': [
			'warn',
			{
				selector: 'default',
				format: ['camelCase'],
				leadingUnderscore: 'forbid',
				trailingUnderscore: 'forbid'
			},
			{
				selector: 'variable',
				modifiers: ['global', 'const'],
				format: ['camelCase', 'UPPER_CASE']
			},
			{
				selector: 'parameter',
				modifiers: ['unused'],
				format: ['camelCase'],
				leadingUnderscore: 'require',
				trailingUnderscore: 'allow'
			},
			{
				selector: 'memberLike',
				modifiers: ['private'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'memberLike',
				modifiers: ['protected'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'typeLike',
				format: ['PascalCase']
			},
			{
				// for non-exported functions
				selector: 'function',
				modifiers: ['global'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'function',
				modifiers: ['exported', 'global'],
				format: ['camelCase'],
				leadingUnderscore: 'forbid'
			}
		],
		'@typescript-eslint/no-import-type-side-effects': 'error',
		'@typescript-eslint/no-require-imports': 'error',
		'@typescript-eslint/no-unnecessary-qualifier': 'error',
		'@typescript-eslint/no-unsafe-unary-minus': 'error',
		'no-unused-vars': 'off',
		'@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
		'@typescript-eslint/no-useless-empty-export': 'error',
		'@typescript-eslint/prefer-enum-initializers': 'error',
		'@typescript-eslint/prefer-readonly': 'error',
		// '@typescript-eslint/prefer-readonly-parameter-types': 'error',
		'@typescript-eslint/prefer-regexp-exec': 'error',
		'@typescript-eslint/promise-function-async': 'error',
		'@typescript-eslint/require-array-sort-compare': 'error',
		'@typescript-eslint/switch-exhaustiveness-check': 'error',

		'@typescript-eslint/non-nullable-type-assertion-style': 'off',
		'@typescript-eslint/unbound-method': 'off'
	},
	overrides: [
		{
			files: ['*.svelte'],
			parser: 'svelte-eslint-parser',
			parserOptions: { parser: '@typescript-eslint/parser' },
			rules: {
				'no-trailing-spaces': 'off',
				'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
			}
		},
		{
			files: ['*.js', '*.cjs'],
			rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
		},
		{
			files: ['*.cjs'],
			rules: { '@typescript-eslint/no-require-imports': 'off' }
		},
		{
			files: ['./*.config.*', '.eslintrc.cjs'],
			rules: { '@typescript-eslint/naming-convention': 'off' }
		}
	]
};
```

:::

このように`overrides`している項目が複数あると flat config によって設定ファイルの記述が簡潔になることが期待できます。

### §1.1　全体に適用するオプションの移行

はじめに、旧設定方式における`ignorePatterns`のような全体に作用するオプションを移行します。

- `root`オプションはなくなりました
- `reportUnusedDisableDirectives`（と`noInlineConfig`）オプションは`linterOptions`にまとめられました
- `ignorePatterns`オプションは`ignores`オプションに名前を変え、`.eslintignore`ファイルは廃止されました
- 全体に適用したい`ignores`オプションは、途中のプラグインの設定等で上書き (overrides) されてしまう可能性があるため、配列の末尾に追加するとよいです

:::details §1.1　diff .eslintrc.cjs → eslint.config.js

`eslint-disable`コメントは、[`@types/node`](https://www.npmjs.com/package/@types/node)パッケージによって`process.env`に型がついたため不要になりました。

```diff js:.eslintrc.cjs → eslint.config.js
-// eslint-disable-next-line @typescript-eslint/no-unsafe-member-access
 const isProduction = () => process.env.NODE_ENV === 'production';

-/** @type {import('eslint').Linter.Config} */
-module.exports = {
+/** @type {import('eslint').Linter.FlatConfig} */
+export default [
-	root: true,
-	reportUnusedDisableDirectives: true,
-	ignorePatterns: ['.svelte-kit/', /* 省略 */, 'node_modules/'],
 	plugins: ['@typescript-eslint'],
 	// 省略
-};
+	{ linterOptions: { reportUnusedDisableDirectives: true } },
+	{ ignores: ['.svelte-kit/', /* 省略 */, 'node_modules/'] }
+];
```

:::

### §1.2　プラグインおよび Sharable Configs の移行

続けて各種プラグイン、および、旧設定ファイルにおいては`extends`プロパティで指定していた sharable config の設定を移行していきます。

- `files`オプションで各設定の適用範囲を設定すべきです
    - 例えば、`tsEslint.strictTypeChecked`では`files`に`['**/*.ts', '**/*.tsx', '**/*.mts', '**/*.cts']`のみが設定されてしまうので、所望のファイルに対して適切に設定が反映されるように**必ず`files`オプションを設定する**ようにします
- `files`を指定する際は、glob syntax (`*.ts, *.tsx`など) と minimatch syntax (`**/*.ts, **/*.tsx`など) のどちらを用いるかを統一したほうがよいでしょう
- Sharable configs は JavaScript modules としてインポートする形式になりました。これにより、`eslint-plugin-*`などの prefix は特別なものではなくなりました
- `eslint:recommended`などの predefined config は[`@eslint/js`](https://www.npmjs.com/package/@eslint/js)パッケージに移行しました
    - `@eslint/js@8.57.0`には型定義がない（`node_modules/.cache/**/@eslint/js`に自動生成される）ため、[`@types/eslint__js`](https://www.npmjs.com/package/@types/eslint__js)パッケージをインストールして ESLint に教えてあげます

        ```sh
        npm i -D @types/eslint__js
        ```

- `plugin:@typescript-eslint/strict-type-checked`などの typescript-eslint 由来の sharable configs は`.configs.strictTypeChecked`などとして`typescript-eslint`のメンバとしてエクスポートされることになりました（参考：[Shared Config | typescript-eslint](https://typescript-eslint.io/users/configs)）
- `plugin:svelte/all`などの eslint-plugin-svelte 由来の sharable configs は`configs['flat/all]`などとして`eslint-plugin-svelte`のメンバとしてエクスポートされることになりました（参考：[User Guide](https://sveltejs.github.io/eslint-plugin-svelte/user-guide/#new-config-eslint-config-js)）
- 前述したように flat config では配列内の後ろの要素で前の要素の設定を上書きするため、依然として設定の順番に気をつける必要はあります

:::details §1.2　diff .eslintrc.cjs → eslint.config.js

私のプロジェクトには`.eslintrc.cjs`を除いて`*.cjs`ファイルはなく、`[*.js, *.ts, *.svelte]`ファイルを ESLint できれば十分なため、`files`はそのように設定しました。

```diff js:.eslintrc.cjs → eslint.config.js
+import js from '@eslint/js';
+import tsEslint from 'typescript-eslint';
+import prettier from 'eslint-config-prettier';
+import svelte from 'eslint-plugin-svelte';

 const isProduction = () => process.env.NODE_ENV === 'production';

+/** @type {import('eslint').Linter.FlatConfigFileSpec[]} */
+const files = ['**/*.js', '**/*.ts', '**/*.svelte'];

 /** @type {import('typescript-eslint').Config} */
 export default [
-	plugins: ['@typescript-eslint'],
-	extends: [
-		'eslint:recommended',
-		'plugin:@typescript-eslint/strict-type-checked',
-		'plugin:@typescript-eslint/stylistic-type-checked',
-		'prettier',
-		'plugin:svelte/all',
-		'plugin:svelte/prettier'
-	],
+	...[
+		js.configs.recommended,
+		...tsEslint.configs.strictTypeChecked,
+		...tsEslint.configs.stylisticTypeChecked,
+		prettier
+	].map((config) => ({ ...config, files })),
+	...[
+		...svelte.configs['flat/all'],
+		...svelte.configs['flat/prettier']
+	].map((config) => ({ ...config, files: ['**/*.svelte'] })),
 	parser: '@typescript-eslint/parser',
 	// 省略
 	{ ignores: ['.svelte-kit/', /* 省略 */, 'node_modules/'] }
 ];
```

:::

### §1.3　パーサーオプションの移行

パーサー周りの設定を移行していきます。

- プラグインの場合と同様に**必ず`files`オプションで各設定の適用範囲を設定します**
- `parser`や`parserOptions`、そして`env`オプションは`languageOptions`にまとめられ、`env`オプションは`globals`に名前を変えました
- `languageOptions.globals`は`env`とは違い、`browser`や`node`といった多数の設定をまとめたプロパティは廃止されました。[`globals`ライブラリ](https://www.npmjs.com/package/globals)を使うとよいらしいです
- 前述したように`overrides`オプションはなくなり、flat config では配列内の後ろの要素で前の要素の設定を上書き (`overrides`) するため、依然として設定の順番に気をつける必要はあります

:::details §1.3　diff .eslintrc.cjs → eslint.config.js

[`parserOptions.tsconfigRootDir`](https://typescript-eslint.io/packages/parser/#tsconfigrootdir)を適切に設定しておくことで、TSConfig ([`parserOptions.project`](https://typescript-eslint.io/packages/parser/#project)) を相対パスで指定できるようになります。

```diff js:.eslintrc.cjs → eslint.config.js
 // 省略
+import svelteParser from 'svelte-eslint-parser';
+import globals from 'globals';

 const isProduction = () => process.env.NODE_ENV === 'production';

 /** @type {import('eslint').Linter.FlatConfigFileSpec[]} */
 const files = ['**/*.js', '**/*.ts', '**/*.svelte'];

 /** @type {import('typescript-eslint').Config} */
 export default [
 	...[
 		js.configs.recommended,
 		...tsEslint.configs.strictTypeChecked,
 		...tsEslint.configs.stylisticTypeChecked,
 		prettier
 	].map((config) => ({ ...config, files })),
 	...[
 		...svelte.configs['flat/all'],
 		...svelte.configs['flat/prettier']
 	].map((config) => ({ ...config, files: ['**/*.svelte'] })),
-	parser: '@typescript-eslint/parser',
-	parserOptions: {
-		sourceType: 'module',
-		ecmaVersion: 'latest',
-		project: './tsconfig.eslint.json',
-		extraFileExtensions: ['.svelte']
-	},
-	env: {
-		browser: true,
-		es2023: true,
-		node: true
-	},
+	{
+		files,
+		languageOptions: {
+			parser: tsEslint.parser,
+			parserOptions: {
+				sourceType: 'module',
+				ecmaVersion: 'latest',
+				project: './tsconfig.eslint.json',
+				tsconfigRootDir: import.meta.dirname,
+				extraFileExtensions: ['.svelte']
+			},
+			globals: { ...globals.browser, ...globals.es2021, ...globals.node }
+		}
+	},
 	rules: {
 		// 省略
 	},
 	overrides: [
-		{
-			files: ['*.svelte'],
-			parser: 'svelte-eslint-parser',
-			parserOptions: { parser: '@typescript-eslint/parser' },
-			rules: {
-				'no-trailing-spaces': 'off',
-				'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
-			}
-		},
+	{
+		files: ['**/*.svelte'],
+		languageOptions: {
+			parser: svelteParser,
+			parserOptions: { parser: tsEslint.parser }
+		},
+		rules: {
+			'no-trailing-spaces': 'off',
+			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
+		}
+	},
 	// 省略
 	{ ignores: ['.svelte-kit/', /* 省略 */, 'node_modules/'] }
 ];
```

:::

### §1.4　ルール設定の移行

自分で設定したルールセットを順番に気をつけて移行します。
例えば、Svelte ファイルにのみ適用したいルールセットを JS, TS, Svelte ファイル全体に対して適用したいルールセットの設定のあとの要素にしてしまうと、思った結果になりません。
後ろの要素で上書き (`overrides`) されるためです。

:::details §1.4　diff .eslintrc.cjs → eslint.config.js

私のプロジェクトには`.eslintrc.cjs`を除いて`*.cjs`ファイルはなく、`[*.js, *.ts, *.svelte]`ファイルを ESLint できれば十分なため、`files`はそのように設定しました。
また`eslint.config.js`では`require()`を使わなくなったため、`@typescript-eslint/no-require-imports`ルールを無効化する必要がなくなりました。

```diff js:.eslintrc.cjs → eslint.config.js
 // 省略
 export default [
 	...[
 		js.configs.recommended,
 		...tsEslint.configs.strictTypeChecked,
 		...tsEslint.configs.stylisticTypeChecked,
 		prettier
 	].map((config) => ({ ...config, files })),
 	...[
 		...svelte.configs['flat/all'],
 		...svelte.configs['flat/prettier']
 	].map((config) => ({ ...config, files: ['**/*.svelte'] })),
 	{
 		files,
 		languageOptions: {
 			parser: tsEslint.parser,
 			parserOptions: {
 				sourceType: 'module',
 				ecmaVersion: 'latest',
 				project: './tsconfig.eslint.json',
 				extraFileExtensions: ['.svelte']
 			},
 			globals: { ...globals.browser, ...globals.es2021, ...globals.node }
-		}
+		},
+		rules: {
+			'no-console': isProduction() ? 'error' : 'off',
+			// 省略
+			'@typescript-eslint/unbound-method': 'off'
+		}
 	},
-	rules: {
-		'no-console': isProduction() ? 'error' : 'off',
-		// 省略
-		'@typescript-eslint/unbound-method': 'off'
-	},
-	overrides: [
 	{
 		files: ['**/*.svelte'],
 		languageOptions: {
 			parser: svelteParser,
 			parserOptions: { parser: tsEslint.parser }
 		},
 		rules: {
 			'no-trailing-spaces': 'off',
 			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
 		}
 	},
-		{
-			files: ['*.js', '*.cjs'],
-			rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
-		},
-		{
-			files: ['*.cjs'],
-			rules: { '@typescript-eslint/no-require-imports': 'off' }
-		},
-		{
-			files: ['./*.config.*', '.eslintrc.cjs'],
-			rules: { '@typescript-eslint/naming-convention': 'off' }
-		}
-	]
+	{
+		files: ['**/*.js'],
+		rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
+	},
+	{
+		files: ['**/*.config.*'],
+		rules: { '@typescript-eslint/naming-convention': 'off' }
+	},
 	{ linterOptions: { reportUnusedDisableDirectives: true } },
 	{ ignores: ['.svelte-kit/', /* 省略 */, 'node_modules/'] }
 ];
```

:::

### §1.5　`files`ごとに設定を変数にまとめる

Flat config では、旧設定方式における`parser`や`extends`の名前解決を ESLint が行わずともよくなったため、すべてをモジュールとして設定を変数やファイルに分割したりすることができるようになりました。
すなわち、**リファクタリングです！**
Flat config ではリファクタリングができるようになりました。

後々設定を変更しやすいように、同じ`files`の設定群でまとめておきます。

#### Step 0.　リファクタリング前の`eslint.config.js`の確認

前節 §1.1.3 までで、`.eslintrc.cjs`は`eslint.config.js`として flat config に移行されました。
ここまでの結果を確認しておきましょう。

:::details eslint.config.jsはこんな感じ

```js:eslint.config.js
import js from '@eslint/js';
import tsEslint from 'typescript-eslint';
import prettier from 'eslint-config-prettier';
import svelte from 'eslint-plugin-svelte';
import svelteParser from 'svelte-eslint-parser';
import globals from 'globals';

const isProduction = () => process.env.NODE_ENV === 'production';

/** @type {import('eslint').Linter.FlatConfigFileSpec[]} */
const files = ['**/*.js', '**/*.ts', '**/*.svelte'];

/** @type {import('typescript-eslint').Config} */
export default [
	...[
		js.configs.recommended,
		...tsEslint.configs.strictTypeChecked,
		...tsEslint.configs.stylisticTypeChecked,
		prettier
	].map((config) => ({ ...config, files })),
	...[
		...svelte.configs['flat/all'],
		...svelte.configs['flat/prettier']
	].map((config) => ({ ...config, files: ['**/*.svelte'] })),
	{
		files,
		languageOptions: {
			parser: tsEslint.parser,
			parserOptions: {
				sourceType: 'module',
				ecmaVersion: 'latest',
				project: './tsconfig.eslint.json',
				extraFileExtensions: ['.svelte']
			},
			globals: { ...globals.browser, ...globals.es2021, ...globals.node }
		},
		rules: {
			'no-console': isProduction() ? 'error' : 'off',

			eqeqeq: ['error', 'always', { null: 'ignore' }],
			'no-duplicate-imports': ['error', { includeExports: true }],
			'no-restricted-imports': [
				'error',
				{ patterns: [{ group: ['../*', 'src/lib/*'], message: 'use `$lib/*` instead' }] }
			],
			'no-trailing-spaces': 'warn',
			'no-unused-expressions': 'error',
			'no-var': 'error',
			'prefer-const': 'error',

			'svelte/no-reactive-reassign': ['error', { props: true }],
			'svelte/block-lang': ['error', { script: 'ts', style: null }],
			'svelte/no-inline-styles': 'off',
			'svelte/no-unused-class-name': 'warn',
			'svelte/no-useless-mustaches': 'warn',
			'svelte/no-restricted-html-elements': 'off',
			'svelte/require-optimized-style-attribute': 'warn',
			'svelte/sort-attributes': 'off',
			'svelte/experimental-require-slot-types': 'off',
			'svelte/experimental-require-strict-events': 'off',

			'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
			'@typescript-eslint/consistent-type-definitions': ['error', 'type'],
			'@typescript-eslint/consistent-type-exports': 'error',
			'@typescript-eslint/consistent-type-imports': 'error',
			'@typescript-eslint/explicit-function-return-type': 'error',
			'@typescript-eslint/explicit-member-accessibility': ['warn', { accessibility: 'no-public' }],
			'@typescript-eslint/member-delimiter-style': 'warn',
			'@typescript-eslint/method-signature-style': 'error',
			camelcase: 'off',
			'@typescript-eslint/naming-convention': [
				'warn',
				{
					selector: 'default',
					format: ['camelCase'],
					leadingUnderscore: 'forbid',
					trailingUnderscore: 'forbid'
				},
				{
					selector: 'variable',
					modifiers: ['global', 'const'],
					format: ['camelCase', 'UPPER_CASE']
				},
				{
					selector: 'parameter',
					modifiers: ['unused'],
					format: ['camelCase'],
					leadingUnderscore: 'require',
					trailingUnderscore: 'allow'
				},
				{
					selector: 'memberLike',
					modifiers: ['private'],
					format: ['camelCase'],
					leadingUnderscore: 'require'
				},
				{
					selector: 'memberLike',
					modifiers: ['protected'],
					format: ['camelCase'],
					leadingUnderscore: 'require'
				},
				{
					selector: 'typeLike',
					format: ['PascalCase']
				},
				{
					// for non-exported functions
					selector: 'function',
					modifiers: ['global'],
					format: ['camelCase'],
					leadingUnderscore: 'require'
				},
				{
					selector: 'function',
					modifiers: ['exported', 'global'],
					format: ['camelCase'],
					leadingUnderscore: 'forbid'
				}
			],
			'@typescript-eslint/no-import-type-side-effects': 'error',
			'@typescript-eslint/no-require-imports': 'error',
			'@typescript-eslint/no-unnecessary-qualifier': 'error',
			'@typescript-eslint/no-unsafe-unary-minus': 'error',
			'no-unused-vars': 'off',
			'@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
			'@typescript-eslint/no-useless-empty-export': 'error',
			'@typescript-eslint/prefer-enum-initializers': 'error',
			'@typescript-eslint/prefer-readonly': 'error',
			// '@typescript-eslint/prefer-readonly-parameter-types': 'error',
			'@typescript-eslint/prefer-regexp-exec': 'error',
			'@typescript-eslint/promise-function-async': 'error',
			'@typescript-eslint/require-array-sort-compare': 'error',
			'@typescript-eslint/switch-exhaustiveness-check': 'error',

			'@typescript-eslint/non-nullable-type-assertion-style': 'off',
			'@typescript-eslint/unbound-method': 'off'
		}
	},
	{
		files: ['**/*.svelte'],
		languageOptions: {
			parser: svelteParser,
			parserOptions: { parser: tsEslint.parser }
		},
		rules: {
			'no-trailing-spaces': 'off',
			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
		}
	},
	{
		files: ['**/*.js'],
		rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
	},
	{
		files: ['**/*.config.*'],
		rules: { '@typescript-eslint/naming-convention': 'off' }
	},
	{ linterOptions: { reportUnusedDisableDirectives: true } },
	{
		ignores: [
			'.svelte-kit/',
			'.vercel/', // adapter-vercel output dir
			'.vercel_build_output/', // old output dir
			'static/',
			'build/',
			'coverage/', // vitest coverage
			'vite.config.ts.timestamp*', // vite temp file
			'node_modules/'
		]
	}
];
```

:::

旧設定方式とそこまで変わっているわけではありません。
Flat config 移行前は気にしていませんでしが、これは関心 (`files`) が分離されていて読みにくいコードです。

#### Step 1.　JS, TS, Svelte 共通の設定を１か所にまとめる

JS, TS, Svelte 共通の設定を`defaultConfig`として1か所にまとめます。

- プラグインの設定に上書きされないように、`languageOptions`等の自分で設定するものは配列の末尾にします

:::details Step 1　diff eslint.config.js

`svelte/*`系のルールは Svelte の設定のほうに移動しました。
また flat config 配列の型定義を各変数ごとに使うので、`FlatConfig`型として`@typedef`しました。

```diff js:eslint.config.js
 import js from '@eslint/js';
 import tsEslint from 'typescript-eslint';
 import prettier from 'eslint-config-prettier';
 import svelte from 'eslint-plugin-svelte';
 import svelteParser from 'svelte-eslint-parser';
 import globals from 'globals';
+
+/** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */

 const isProduction = () => process.env.NODE_ENV === 'production';

-/** @type {import('eslint').Linter.FlatConfigFileSpec[]} */
-const files = ['**/*.js', '**/*.ts', '**/*.svelte'];
+/** @type {FlatConfig[]} */
+const defaultConfigWithoutExtensions = [
+	js.configs.recommended,
+	...tsEslint.configs.strictTypeChecked,
+	...tsEslint.configs.stylisticTypeChecked,
+	prettier,
+	{
+		languageOptions: {
+			parser: tsEslint.parser,
+			parserOptions: {
+				sourceType: 'module',
+				ecmaVersion: 2023,
+				project: './tsconfig.eslint.json',
+				tsconfigRootDir: import.meta.dirname,
+				extraFileExtensions: ['.svelte']
+			},
+			globals: { ...globals.browser, ...globals.es2021, ...globals.node }
+		},
+		rules: {
+			'no-console': isProduction() ? 'error' : 'off',
+			// 省略
+			'prefer-const': 'error',
+
+			'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
+			// 省略
+			'@typescript-eslint/unbound-method': 'off'
+		}
+	}
+];

-/** @type {import('typescript-eslint').Config} */
+/** @type {FlatConfig[]} */
 export default [
--	...[
--		js.configs.recommended,
--		...tsEslint.configs.strictTypeChecked,
--		...tsEslint.configs.stylisticTypeChecked,
--		prettier
--	].map((config) => ({ ...config, files })),
+	...defaultConfigWithoutExtensions.map(
+		(config) => ({ ...config, files: ['**/*.js', '**/*.ts', '**/*.svelte'] })
+	),
 	...[
 		...svelte.configs['flat/all'],
 		...svelte.configs['flat/prettier']
 	].map((config) => ({ ...config, files: ['**/*.svelte'] })),
-	{
-		files,
-		languageOptions: {
-			parser: tsEslint.parser,
-			parserOptions: {
-				sourceType: 'module',
-				ecmaVersion: 'latest',
-				project: './tsconfig.eslint.json',
-				extraFileExtensions: ['.svelte']
-			},
-			globals: { ...globals.browser, ...globals.es2021, ...globals.node }
-		},
-		rules: {
-			'no-console': isProduction() ? 'error' : 'off',
-			// 省略
-			'prefer-const': 'error',
-
-			'svelte/no-reactive-reassign': ['error', { props: true }],
-			// 省略
-			'svelte/experimental-require-strict-events': 'off',
-
-			'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
-			// 省略
-			'@typescript-eslint/unbound-method': 'off'
-		}
-	},
 	{
 		files: ['**/*.svelte'],
 		languageOptions: {
 			parser: svelteParser,
 			parserOptions: { parser: tsEslint.parser }
 		},
 		rules: {
+			'svelte/no-reactive-reassign': ['error', { props: true }],
+			// 省略
+			'svelte/experimental-require-strict-events': 'off',
 			'no-trailing-spaces': 'off',
 			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
 		}
 	},
 	{
 		files: ['**/*.js'],
 		rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
 	},
 	{
 		files: ['**/*.config.*'],
 		rules: { '@typescript-eslint/naming-convention': 'off' }
 	},
 	{ linterOptions: { reportUnusedDisableDirectives: true } },
 	{ ignores: ['.svelte-kit/', /* 省略 */ 'node_modules/'] }
 ];
```

:::

#### Step 2　Svelte の設定を一つの配列にまとめる

Svelte の設定を`svelteConfig`としてまとめます。

- プラグインの設定に上書きされないように、`languageOptions`等の自分で設定するものは配列の末尾にします
- `eslint-plugin-svelte`は`module 'eslint' { namespace Linter { interface RulesRecord`を上書きしているため、JSDoc を設定することで`svelte/*`ルールに型補完が利くようになります（参考：[feat: add rule types by xiBread・Pull Request #735・sveltejs/eslint-plugin-svelte](https://github.com/sveltejs/eslint-plugin-svelte/pull/735)）

:::details Step 2　diff eslint.config.js

```diff js:eslint.config.js
 import js from '@eslint/js';
 import tsEslint from 'typescript-eslint';
 import prettier from 'eslint-config-prettier';
 import svelte from 'eslint-plugin-svelte';
 import svelteParser from 'svelte-eslint-parser';
 import globals from 'globals';

 /** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */

 const isProduction = () => process.env.NODE_ENV === 'production';

 /** @type {FlatConfig[]} */
 const defaultConfigWithoutExtensions = [
 	// 省略
 ];
+
+/** @type {FlatConfig[]} */
+const svelteConfigWithoutExtensions = [
+	...svelte.configs['flat/all'],
+	...svelte.configs['flat/prettier'],
+	{
+		languageOptions: {
+			parser: svelteParser,
+			parserOptions: { parser: tsEslint.parser }
+		},
+		/** @type {import('eslint').Linter.RulesRecord} */
+		rules: {
+			'svelte/no-reactive-reassign': ['error', { props: true }],
+			// 省略
+			'svelte/experimental-require-strict-events': 'off',
+			'no-trailing-spaces': 'off',
+			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
+		}
+	}
+].map((config) => ({ ...config, files: ['**/*.svelte'] }));

 /** @type {FlatConfig[]} */
 export default [
 	...defaultConfigWithoutExtensions.map(
 		(config) => ({ ...config, files: ['**/*.js', '**/*.ts', '**/*.svelte'] })
 	),
-	...[
-		...svelte.configs['flat/all'],
-		...svelte.configs['flat/prettier']
-	].map((config) => ({ ...config, files: ['**/*.svelte'] })),
+	...svelteConfigWithoutExtensions.map(
+		(config) => ({ ...config, files: ['**/*.svelte'] })
+	),
-	{
-		files: ['**/*.svelte'],
-		languageOptions: {
-			parser: svelteParser,
-			parserOptions: { parser: tsEslint.parser }
-		},
-		rules: {
-			'svelte/no-reactive-reassign': ['error', { props: true }],
-			// 省略
-			'svelte/experimental-require-strict-events': 'off',
-			'no-trailing-spaces': 'off',
-			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
-		}
-	},
 	{
 		files: ['**/*.js'],
 		rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
 	},
 	{
 		files: ['**/*.config.*'],
 		rules: { '@typescript-eslint/naming-convention': 'off' }
 	},
 	{ linterOptions: { reportUnusedDisableDirectives: true } },
 	{ ignores: ['.svelte-kit/', /* 省略 */ 'node_modules/'] }
 ];
```

:::

#### Step 3　その他の設定もまとめる

見た目を統一するため、ついでに他の設定も`files`ごとにまとめておきます。

- 再三になりますが、設定の順番には十分気を付けましょう

:::details Step 3　diff eslint.config.js

```diff js:eslint.config.js
 import js from '@eslint/js';
 import tsEslint from 'typescript-eslint';
 import prettier from 'eslint-config-prettier';
 import svelte from 'eslint-plugin-svelte';
 import svelteParser from 'svelte-eslint-parser';
 import globals from 'globals';

 /** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */

 const isProduction = () => process.env.NODE_ENV === 'production';

 /** @type {FlatConfig[]} */
 const defaultConfigWithoutExtensions = [
 	// 省略
 ];

 /** @type {FlatConfig[]} */
 const svelteConfigWithoutExtensions = [
 	// 省略
 ];
+
+/** @type {FlatConfig} */
+const jsConfig = {
+	files: ['**/*.js'],
+	rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
+};
+
+/** @type {FlatConfig} */
+const configConfig = {
+	files: ['**/*.config.*'],
+	rules: { '@typescript-eslint/naming-convention': 'off' }
+};

 /** @type {FlatConfig[]} */
 export default [
 	...defaultConfigWithoutExtensions.map(
 		(config) => ({ ...config, files: ['**/*.js', '**/*.ts', '**/*.svelte'] })
 	),
 	...svelteConfigWithoutExtensions.map(
 		(config) => ({ ...config, files: ['**/*.svelte'] })
 	),
-	{
-		files: ['**/*.js'],
-		rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
-	},
+	jsConfig,
-	{
-		files: ['**/*.config.*'],
-		rules: { '@typescript-eslint/naming-convention': 'off' }
-	},
+	configConfig,
 	{ linterOptions: { reportUnusedDisableDirectives: true } },
 	{ ignores: ['.svelte-kit/', /* 省略 */ 'node_modules/'] }
 ];
```

:::

### §1.6　移行後のファイルの確認

[ESLint 公式のマイグレーションガイド](https://eslint.org/docs/v8.x/use/configure/migration-guide)にしたがって移行した後の設定ファイルはこのようになりました。

:::details §1.6　eslint.config.js

```js:eslint.config.js
import js from '@eslint/js';
import tsEslint from 'typescript-eslint';
import prettier from 'eslint-config-prettier';
import svelte from 'eslint-plugin-svelte';
import svelteParser from 'svelte-eslint-parser';
import globals from 'globals';

/** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */

const isProduction = () => process.env.NODE_ENV === 'production';

/** @type {FlatConfig[]} */
const defaultConfigWithoutExtensions = [
	js.configs.recommended,
	...tsEslint.configs.strictTypeChecked,
	...tsEslint.configs.stylisticTypeChecked,
	prettier,
	{
		languageOptions: {
			parser: tsEslint.parser,
			parserOptions: {
				sourceType: 'module',
				ecmaVersion: 2023,
				project: './tsconfig.eslint.json',
				tsconfigRootDir: import.meta.dirname,
				extraFileExtensions: ['.svelte']
			},
			globals: { ...globals.browser, ...globals.es2021, ...globals.node }
		},
		rules: {
			'no-console': isProduction() ? 'error' : 'off',

			eqeqeq: ['error', 'always', { null: 'ignore' }],
			'no-duplicate-imports': ['error', { includeExports: true }],
			'no-restricted-imports': [
				'error',
				{ patterns: [{ group: ['../*', 'src/lib/*'], message: 'use `$lib/*` instead' }] }
			],
			'no-trailing-spaces': 'warn',
			'no-unused-expressions': 'error',
			'no-var': 'error',
			'prefer-const': 'error',

			'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
			'@typescript-eslint/consistent-type-definitions': ['error', 'type'],
			'@typescript-eslint/consistent-type-exports': 'error',
			'@typescript-eslint/consistent-type-imports': 'error',
			'@typescript-eslint/explicit-function-return-type': 'error',
			'@typescript-eslint/explicit-member-accessibility': ['warn', { accessibility: 'no-public' }],
			'@typescript-eslint/member-delimiter-style': 'warn',
			'@typescript-eslint/method-signature-style': 'error',
			camelcase: 'off',
			'@typescript-eslint/naming-convention': [
				'warn',
				{
					selector: 'default',
					format: ['camelCase'],
					leadingUnderscore: 'forbid',
					trailingUnderscore: 'forbid'
				},
				{
					selector: 'variable',
					modifiers: ['global', 'const'],
					format: ['camelCase', 'UPPER_CASE']
				},
				{
					selector: 'parameter',
					modifiers: ['unused'],
					format: ['camelCase'],
					leadingUnderscore: 'require',
					trailingUnderscore: 'allow'
				},
				{
					selector: 'memberLike',
					modifiers: ['private'],
					format: ['camelCase'],
					leadingUnderscore: 'require'
				},
				{
					selector: 'memberLike',
					modifiers: ['protected'],
					format: ['camelCase'],
					leadingUnderscore: 'require'
				},
				{
					selector: 'typeLike',
					format: ['PascalCase']
				},
				{
					// for non-exported functions
					selector: 'function',
					modifiers: ['global'],
					format: ['camelCase'],
					leadingUnderscore: 'require'
				},
				{
					selector: 'function',
					modifiers: ['exported', 'global'],
					format: ['camelCase'],
					leadingUnderscore: 'forbid'
				}
			],
			'@typescript-eslint/no-import-type-side-effects': 'error',
			'@typescript-eslint/no-require-imports': 'error',
			'@typescript-eslint/no-unnecessary-qualifier': 'error',
			'@typescript-eslint/no-unsafe-unary-minus': 'error',
			'no-unused-vars': 'off',
			'@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
			'@typescript-eslint/no-useless-empty-export': 'error',
			'@typescript-eslint/prefer-enum-initializers': 'error',
			'@typescript-eslint/prefer-readonly': 'error',
			// '@typescript-eslint/prefer-readonly-parameter-types': 'error',
			'@typescript-eslint/prefer-regexp-exec': 'error',
			'@typescript-eslint/promise-function-async': 'error',
			'@typescript-eslint/require-array-sort-compare': 'error',
			'@typescript-eslint/switch-exhaustiveness-check': 'error',

			'@typescript-eslint/non-nullable-type-assertion-style': 'off',
			'@typescript-eslint/unbound-method': 'off'
		}
	}
];

/** @type {FlatConfig[]} */
const svelteConfigWithoutExtensions = [
	...svelte.configs['flat/all'],
	...svelte.configs['flat/prettier'],
	{
		languageOptions: {
			parser: svelteParser,
			parserOptions: { parser: tsEslint.parser }
		},
		/** @type {import('eslint').Linter.RulesRecord} */
		rules: {
			'svelte/no-reactive-reassign': ['error', { props: true }],
			'svelte/block-lang': ['error', { script: 'ts', style: null }],
			'svelte/no-inline-styles': 'off',
			'svelte/no-unused-class-name': 'warn',
			'svelte/no-useless-mustaches': 'warn',
			'svelte/no-restricted-html-elements': 'off',
			'svelte/require-optimized-style-attribute': 'warn',
			'svelte/sort-attributes': 'off',
			'svelte/experimental-require-slot-types': 'off',
			'svelte/experimental-require-strict-events': 'off',
			'no-trailing-spaces': 'off',
			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
		}
	}
];

/** @type {FlatConfig} */
const jsConfig = {
	files: ['**/*.js'],
	rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
};

/** @type {FlatConfig} */
const configConfig = {
	files: ['**/*.config.*'],
	rules: { '@typescript-eslint/naming-convention': 'off' }
};

/** @type {FlatConfig[]} */
export default [
	...defaultConfigWithoutExtensions.map(
		(config) => ({ ...config, files: ['**/*.js', '**/*.ts', '**/*.svelte'] })
	),
	...svelteConfigWithoutExtensions.map(
		(config) => ({ ...config, files: ['**/*.svelte'] })
	),
	jsConfig,
	configConfig,
	{ linterOptions: { reportUnusedDisableDirectives: true } },
	{
		ignores: [
			'.svelte-kit/',
			'.vercel/', // adapter-vercel output dir
			'.vercel_build_output/', // old output dir
			'static/',
			'build/',
			'coverage/', // vitest coverage
			'vitest.config.ts.timestamp*', // vite temp files
			'node_modules/'
		]
	}
];
```

:::

`files`ごとに設定がまとまっていて、どのルールがどのファイルに適用されるのかがわかりやすくなった気がします。

## §2　typescript-eslint のヘルパー関数 `config(...)` を使って Flat Config に移行する

§1 のまま終わってもよいのですが、`defaultConfig`とその`files`設定の間に行数が空いてしまうのも、設定ファイルに`.map()`が出てくるのも複雑な気がします。
そういった問題に対処するため、typescript-eslint は[ヘルパー関数`.config()`](https://typescript-eslint.io/packages/typescript-eslint#config)を提供しています。

これを使ってさらに改善してみました。

::::details §2　diff/after eslint.config.js

`defaultConfig`の`extends`オプションに JSDoc 型定義を書いていますが、これは`eslint-config-prettier`の型定義がない（`node_modules/.cache/**`に自動生成される）ためです。
[`@types/eslint-config-prettier`](https://www.npmjs.com/package/@types/eslint-config-prettier)パッケージをインストールして ESLint に教えてあげてもよいでしょう。

```sh
npm i -D @types/eslint-config-prettier
```

:::details diff eslint.config.js

```diff js:eslint.config.js
 import js from '@eslint/js';
 import tsEslint from 'typescript-eslint';
 import prettier from 'eslint-config-prettier';
 import svelte from 'eslint-plugin-svelte';
 import svelteParser from 'svelte-eslint-parser';
 import globals from 'globals';

-/** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */
-
 const isProduction = () => process.env.NODE_ENV === 'production';

-/** @type {FlatConfig[]} */
-const defaultConfigWithoutExtensions = [
-	js.configs.recommended,
-	...tsEslint.configs.strictTypeChecked,
-	...tsEslint.configs.stylisticTypeChecked,
-	prettier,
+const defaultConfig = tsEslint.config({
+	files: ['**/*.js', '**/*.ts', '**/*.svelte'],
+	/** @type {import('typescript-eslint').ConfigWithExtends['extends']} */
+	extends: [
+		js.configs.recommended,
+		...tsEslint.configs.strictTypeChecked,
+		...tsEslint.configs.stylisticTypeChecked,
+		prettier
+	],
-	{
-		languageOptions: {
-			// 省略
-		},
-		rules: {
-			'no-console': isProduction() ? 'error' : 'off',
-			// 省略
-			'@typescript-eslint/unbound-method': 'off'
-		}
-	}
+	languageOptions: {
+		// 省略
+	},
+	rules: {
+		'no-console': isProduction() ? 'error' : 'off',
+		// 省略
+		'@typescript-eslint/unbound-method': 'off'
+	}
-];
+});

-/** @type {FlatConfig[]} */
-const svelteConfigWithoutExtensions = [
-	...svelte.configs['flat/all'],
-	...svelte.configs['flat/prettier'],
+const svelteConfig = tsEslint.config({
+	files: ['**/*.svelte'],
+	extends: [
+		...svelte.configs['flat/all'],
+		...svelte.configs['flat/prettier']
+	],
-	{
-		languageOptions: {
-			parser: svelteParser,
-			parserOptions: { parser: tsEslint.parser }
-		},
-		/** @type {import('eslint').Linter.RulesRecord} */
-		rules: {
-			'svelte/no-reactive-reassign': ['error', { props: true }],
-			// 省略
-			'no-trailing-spaces': 'off',
-			'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
-		}
-	}
+	languageOptions: {
+		parser: svelteParser,
+		parserOptions: { parser: tsEslint.parser }
+	},
+	/** @type {import('eslint').Linter.RulesRecord} */
+	rules: {
+		'svelte/no-reactive-reassign': ['error', { props: true }],
+		// 省略
+		'no-trailing-spaces': 'off',
+		'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
+	}
-];
+});

+/** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */
+
 /** @type {FlatConfig} */
 const jsConfig = {
 	// 省略
 };

 /** @type {FlatConfig} */
 const configConfig = {
 	// 省略
 };

 /** @type {FlatConfig[]} */
 export default [
-	...defaultConfigWithoutExtensions.map(
-		(config) => ({ ...config, files: ['**/*.js', '**/*.ts', '**/*.svelte'] })
-	),
+	...defaultConfig,
-	...svelteConfigWithoutExtensions.map(
-		(config) => ({ ...config, files: ['**/*.svelte'] })
-	),
+	...svelteConfig,
 	jsConfig,
 	configConfig,
 	{ linterOptions: { reportUnusedDisableDirectives: true } },
 	{ ignores: ['.svelte-kit/', /* 省略 */ 'node_modules/'] }
 ];
```

:::

:::details ヘルパー関数使用後の eslint.config.js

```js:eslint.config.js
import js from '@eslint/js';
import tsEslint from 'typescript-eslint';
import prettier from 'eslint-config-prettier';
import svelte from 'eslint-plugin-svelte';
import svelteParser from 'svelte-eslint-parser';
import globals from 'globals';

const isProduction = () => process.env.NODE_ENV === 'production';

const defaultConfig = tsEslint.config({
	files: ['**/*.js', '**/*.ts', '**/*.svelte'],
	/** @type {import('typescript-eslint').ConfigWithExtends['extends']} */
	extends: [
		js.configs.recommended,
		...tsEslint.configs.strictTypeChecked,
		...tsEslint.configs.stylisticTypeChecked,
		prettier
	],
	languageOptions: {
		parser: tsEslint.parser,
		parserOptions: {
			sourceType: 'module',
			ecmaVersion: 2023,
			project: './tsconfig.eslint.json',
			tsconfigRootDir: import.meta.dirname,
			extraFileExtensions: ['.svelte']
		},
		globals: { ...globals.browser, ...globals.es2021, ...globals.node }
	},
	rules: {
		'no-console': isProduction() ? 'error' : 'off',

		eqeqeq: ['error', 'always', { null: 'ignore' }],
		'no-duplicate-imports': ['error', { includeExports: true }],
		'no-restricted-imports': [
			'error',
			{ patterns: [{ group: ['../*', 'src/lib/*'], message: 'use `$lib/*` instead' }] }
		],
		'no-trailing-spaces': 'warn',
		'no-unused-expressions': 'error',
		'no-var': 'error',
		'prefer-const': 'error',

		'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
		'@typescript-eslint/consistent-type-definitions': ['error', 'type'],
		'@typescript-eslint/consistent-type-exports': 'error',
		'@typescript-eslint/consistent-type-imports': 'error',
		'@typescript-eslint/explicit-function-return-type': 'error',
		'@typescript-eslint/explicit-member-accessibility': ['warn', { accessibility: 'no-public' }],
		'@typescript-eslint/member-delimiter-style': 'warn',
		'@typescript-eslint/method-signature-style': 'error',
		camelcase: 'off',
		'@typescript-eslint/naming-convention': [
			'warn',
			{
				selector: 'default',
				format: ['camelCase'],
				leadingUnderscore: 'forbid',
				trailingUnderscore: 'forbid'
			},
			{
				selector: 'variable',
				modifiers: ['global', 'const'],
				format: ['camelCase', 'UPPER_CASE']
			},
			{
				selector: 'parameter',
				modifiers: ['unused'],
				format: ['camelCase'],
				leadingUnderscore: 'require',
				trailingUnderscore: 'allow'
			},
			{
				selector: 'memberLike',
				modifiers: ['private'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'memberLike',
				modifiers: ['protected'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'typeLike',
				format: ['PascalCase']
			},
			{
				// for non-exported functions
				selector: 'function',
				modifiers: ['global'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'function',
				modifiers: ['exported', 'global'],
				format: ['camelCase'],
				leadingUnderscore: 'forbid'
			}
		],
		'@typescript-eslint/no-import-type-side-effects': 'error',
		'@typescript-eslint/no-require-imports': 'error',
		'@typescript-eslint/no-unnecessary-qualifier': 'error',
		'@typescript-eslint/no-unsafe-unary-minus': 'error',
		'no-unused-vars': 'off',
		'@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
		'@typescript-eslint/no-useless-empty-export': 'error',
		'@typescript-eslint/prefer-enum-initializers': 'error',
		'@typescript-eslint/prefer-readonly': 'error',
		// '@typescript-eslint/prefer-readonly-parameter-types': 'error',
		'@typescript-eslint/prefer-regexp-exec': 'error',
		'@typescript-eslint/promise-function-async': 'error',
		'@typescript-eslint/require-array-sort-compare': 'error',
		'@typescript-eslint/switch-exhaustiveness-check': 'error',

		'@typescript-eslint/non-nullable-type-assertion-style': 'off',
		'@typescript-eslint/unbound-method': 'off'
	}
});

const svelteConfig = tsEslint.config({
	files: ['**/*.svelte'],
	extends: [
		...svelte.configs['flat/all'],
		...svelte.configs['flat/prettier']
	],
	languageOptions: {
		parser: svelteParser,
		parserOptions: { parser: tsEslint.parser }
	},
	/** @type {import('eslint').Linter.RulesRecord} */
	rules: {
		'svelte/no-reactive-reassign': ['error', { props: true }],
		'svelte/block-lang': ['error', { script: 'ts', style: null }],
		'svelte/no-inline-styles': 'off',
		'svelte/no-unused-class-name': 'warn',
		'svelte/no-useless-mustaches': 'warn',
		'svelte/no-restricted-html-elements': 'off',
		'svelte/require-optimized-style-attribute': 'warn',
		'svelte/sort-attributes': 'off',
		'svelte/experimental-require-slot-types': 'off',
		'svelte/experimental-require-strict-events': 'off',
		'no-trailing-spaces': 'off',
		'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
	}
});

/** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */

/** @type {FlatConfig} */
const jsConfig = {
	files: ['**/*.js'],
	rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
};

/** @type {FlatConfig} */
const configConfig = {
	files: ['**/*.config.*'],
	rules: { '@typescript-eslint/naming-convention': 'off' }
};

 /** @type {FlatConfig[]} */
 export default [
	...defaultConfig,
	...svelteConfig,
	jsConfig,
	configConfig,
	{ linterOptions: { reportUnusedDisableDirectives: true } },
	{
		ignores: [
			'.svelte-kit/',
			'.vercel/', // adapter-vercel output dir
			'.vercel_build_output/', // old output dir
			'static/',
			'build/',
			'coverage/', // vitest coverage
			'vitest.config.ts.timestamp*', // vite temp files
			'node_modules/'
		]
	}
];
```

:::

::::

対象となる`files`と設定が同じオブジェクトに記述されることで、どのルールがどのファイルに適用されるのかが更にわかりやすくなった気がします。

## §3　移行まとめ

`typescript-eslint`の提供するヘルパー関数`.config()`を使って flat config に移行しました。
どの`files`にどの設定・ルールが適用されるか非常にわかりやすくなったと感じます。
一度 flat config に移行してしまえば、ESLint の設定ファイルに詳しくなくとも設定変更作業ができそうです。

重要な変更点としては、

- 大オブジェクトのプロパティによる設定形式から、`files`ごとに設定オブジェクトを作成して配列にする形式に
    - 旧`overrides`オプションのイメージ
- プラグインの解決を ESLint では行わなくなった
    - ESLint プラグインの命名が自由になりました
    - [`@rushstack/eslint-patch`](https://www.npmjs.com/package/@rushstack/eslint-patch)の modern-module-resolution オプションは不要に
- `.eslintignore`ファイルおよび`ignorePatterns`オプションの廃止
    - `ignores`オプションに一本化（`files`オプションとの競合に注意）
- `languageOptions`の追加
    - `parser`, `parserOptions`, `env`を統合
    - `env`オプションは`languageOptions.globals`になり、`browsers`や`node`などのプロパティによる設定から、[`globals`](https://www.npmjs.com/package/globals)パッケージを使っての設定に
- `linterOptions`の追加
    - `reportUnusedDisableDirectives`, `noInlineConfig`を統合

などでしょうか。

これまでの設定方式では[`@rushstack/eslint-patch`](https://www.npmjs.com/package/@rushstack/eslint-patch)と`overrides`オプション、`require()`/`import()`をフルに使わないとできなかったことが簡潔にできるようになり、黒魔術的な設定ファイルから別れを告げられるようになりました。
個人的には、sharable config をこれまで以上に簡易に作れるようになったのが嬉しいです。

::::details 最後にもう一度移行前と移行後のファイルを並べておきます。

:::details 移行前の設定ファイル .eslintrc.cjs

```js:.eslintrc.cjs
// eslint-disable-next-line @typescript-eslint/no-unsafe-member-access
const isProduction = () => process.env.NODE_ENV === 'production';

/** @type {import('eslint').Linter.Config} */
module.exports = {
	root: true,
	reportUnusedDisableDirectives: true,
	ignorePatterns: [
		'.svelte-kit/',
		'.vercel/', // adapter-vercel output dir
		'.vercel_build_output/', // old output dir
		'static/',
		'build/',
		'coverage/', // vitest coverage
		'vitest.config.ts.timestamp*', // vite temp files
		'node_modules/'
	],
	plugins: ['@typescript-eslint'],
	extends: [
		'eslint:recommended',
		'plugin:@typescript-eslint/strict-type-checked',
		'plugin:@typescript-eslint/stylistic-type-checked',
		'prettier',
		'plugin:svelte/all',
		'plugin:svelte/prettier'
	],
	parser: '@typescript-eslint/parser',
	parserOptions: {
		sourceType: 'module',
		ecmaVersion: 'latest',
		project: './tsconfig.eslint.json',
		extraFileExtensions: ['.svelte']
	},
	env: {
		browser: true,
		es2022: true,
		node: true
	},
	rules: {
		'no-console': isProduction() ? 'error' : 'off',

		eqeqeq: ['error', 'always', { null: 'ignore' }],
		'no-duplicate-imports': ['error', { includeExports: true }],
		'no-restricted-imports': [
			'error',
			{ patterns: [{ group: ['../*', 'src/lib/*'], message: 'use `$lib/*` instead' }] }
		],
		'no-trailing-spaces': 'warn',
		'no-unused-expressions': 'error',
		'no-var': 'error',
		'prefer-const': 'error',

		'svelte/no-reactive-reassign': ['error', { props: true }],
		'svelte/block-lang': ['error', { script: 'ts', style: null }],
		'svelte/no-inline-styles': 'off',
		'svelte/no-unused-class-name': 'warn',
		'svelte/no-useless-mustaches': 'warn',
		'svelte/no-restricted-html-elements': 'off',
		'svelte/require-optimized-style-attribute': 'warn',
		'svelte/sort-attributes': 'off',
		'svelte/experimental-require-slot-types': 'off',
		'svelte/experimental-require-strict-events': 'off',

		'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
		'@typescript-eslint/consistent-type-definitions': ['error', 'type'],
		'@typescript-eslint/consistent-type-exports': 'error',
		'@typescript-eslint/consistent-type-imports': 'error',
		'@typescript-eslint/explicit-function-return-type': 'error',
		'@typescript-eslint/explicit-member-accessibility': ['warn', { accessibility: 'no-public' }],
		'@typescript-eslint/member-delimiter-style': 'warn',
		'@typescript-eslint/method-signature-style': 'error',
		camelcase: 'off',
		'@typescript-eslint/naming-convention': [
			'warn',
			{
				selector: 'default',
				format: ['camelCase'],
				leadingUnderscore: 'forbid',
				trailingUnderscore: 'forbid'
			},
			{
				selector: 'variable',
				modifiers: ['global', 'const'],
				format: ['camelCase', 'UPPER_CASE']
			},
			{
				selector: 'parameter',
				modifiers: ['unused'],
				format: ['camelCase'],
				leadingUnderscore: 'require',
				trailingUnderscore: 'allow'
			},
			{
				selector: 'memberLike',
				modifiers: ['private'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'memberLike',
				modifiers: ['protected'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'typeLike',
				format: ['PascalCase']
			},
			{
				// for non-exported functions
				selector: 'function',
				modifiers: ['global'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'function',
				modifiers: ['exported', 'global'],
				format: ['camelCase'],
				leadingUnderscore: 'forbid'
			}
		],
		'@typescript-eslint/no-import-type-side-effects': 'error',
		'@typescript-eslint/no-require-imports': 'error',
		'@typescript-eslint/no-unnecessary-qualifier': 'error',
		'@typescript-eslint/no-unsafe-unary-minus': 'error',
		'no-unused-vars': 'off',
		'@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
		'@typescript-eslint/no-useless-empty-export': 'error',
		'@typescript-eslint/prefer-enum-initializers': 'error',
		'@typescript-eslint/prefer-readonly': 'error',
		// '@typescript-eslint/prefer-readonly-parameter-types': 'error',
		'@typescript-eslint/prefer-regexp-exec': 'error',
		'@typescript-eslint/promise-function-async': 'error',
		'@typescript-eslint/require-array-sort-compare': 'error',
		'@typescript-eslint/switch-exhaustiveness-check': 'error',

		'@typescript-eslint/non-nullable-type-assertion-style': 'off',
		'@typescript-eslint/unbound-method': 'off'
	},
	overrides: [
		{
			files: ['*.svelte'],
			parser: 'svelte-eslint-parser',
			parserOptions: { parser: '@typescript-eslint/parser' },
			rules: {
				'no-trailing-spaces': 'off',
				'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
			}
		},
		{
			files: ['*.js', '*.cjs'],
			rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
		},
		{
			files: ['*.cjs'],
			rules: { '@typescript-eslint/no-require-imports': 'off' }
		},
		{
			files: ['./*.config.*', '.eslintrc.cjs'],
			rules: { '@typescript-eslint/naming-convention': 'off' }
		}
	]
};
```

:::

:::details flat config 移行後の eslint.config.js

```js:eslint.config.js
import js from '@eslint/js';
import tsEslint from 'typescript-eslint';
import prettier from 'eslint-config-prettier';
import svelte from 'eslint-plugin-svelte';
import svelteParser from 'svelte-eslint-parser';
import globals from 'globals';

const isProduction = () => process.env.NODE_ENV === 'production';

const defaultConfig = tsEslint.config({
	files: ['**/*.js', '**/*.ts', '**/*.svelte'],
	extends: [
		js.configs.recommended,
		...tsEslint.configs.strictTypeChecked,
		...tsEslint.configs.stylisticTypeChecked,
		prettier
	],
	languageOptions: {
		parser: tsEslint.parser,
		parserOptions: {
			sourceType: 'module',
			ecmaVersion: 2023,
			project: './tsconfig.eslint.json',
			tsconfigRootDir: import.meta.dirname,
			extraFileExtensions: ['.svelte']
		},
		globals: { ...globals.browser, ...globals.es2021, ...globals.node }
	},
	rules: {
		'no-console': isProduction() ? 'error' : 'off',

		eqeqeq: ['error', 'always', { null: 'ignore' }],
		'no-duplicate-imports': ['error', { includeExports: true }],
		'no-restricted-imports': [
			'error',
			{ patterns: [{ group: ['../*', 'src/lib/*'], message: 'use `$lib/*` instead' }] }
		],
		'no-trailing-spaces': 'warn',
		'no-unused-expressions': 'error',
		'no-var': 'error',
		'prefer-const': 'error',

		'@typescript-eslint/array-type': ['error', { default: 'array-simple' }],
		'@typescript-eslint/consistent-type-definitions': ['error', 'type'],
		'@typescript-eslint/consistent-type-exports': 'error',
		'@typescript-eslint/consistent-type-imports': 'error',
		'@typescript-eslint/explicit-function-return-type': 'error',
		'@typescript-eslint/explicit-member-accessibility': ['warn', { accessibility: 'no-public' }],
		'@typescript-eslint/member-delimiter-style': 'warn',
		'@typescript-eslint/method-signature-style': 'error',
		camelcase: 'off',
		'@typescript-eslint/naming-convention': [
			'warn',
			{
				selector: 'default',
				format: ['camelCase'],
				leadingUnderscore: 'forbid',
				trailingUnderscore: 'forbid'
			},
			{
				selector: 'variable',
				modifiers: ['global', 'const'],
				format: ['camelCase', 'UPPER_CASE']
			},
			{
				selector: 'parameter',
				modifiers: ['unused'],
				format: ['camelCase'],
				leadingUnderscore: 'require',
				trailingUnderscore: 'allow'
			},
			{
				selector: 'memberLike',
				modifiers: ['private'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'memberLike',
				modifiers: ['protected'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'typeLike',
				format: ['PascalCase']
			},
			{
				// for non-exported functions
				selector: 'function',
				modifiers: ['global'],
				format: ['camelCase'],
				leadingUnderscore: 'require'
			},
			{
				selector: 'function',
				modifiers: ['exported', 'global'],
				format: ['camelCase'],
				leadingUnderscore: 'forbid'
			}
		],
		'@typescript-eslint/no-import-type-side-effects': 'error',
		'@typescript-eslint/no-require-imports': 'error',
		'@typescript-eslint/no-unnecessary-qualifier': 'error',
		'@typescript-eslint/no-unsafe-unary-minus': 'error',
		'no-unused-vars': 'off',
		'@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
		'@typescript-eslint/no-useless-empty-export': 'error',
		'@typescript-eslint/prefer-enum-initializers': 'error',
		'@typescript-eslint/prefer-readonly': 'error',
		// '@typescript-eslint/prefer-readonly-parameter-types': 'error',
		'@typescript-eslint/prefer-regexp-exec': 'error',
		'@typescript-eslint/promise-function-async': 'error',
		'@typescript-eslint/require-array-sort-compare': 'error',
		'@typescript-eslint/switch-exhaustiveness-check': 'error',

		'@typescript-eslint/non-nullable-type-assertion-style': 'off',
		'@typescript-eslint/unbound-method': 'off'
	}
});

const svelteConfig = tsEslint.config({
	files: ['**/*.svelte'],
	extends: [
		...svelte.configs['flat/all'],
		...svelte.configs['flat/prettier']
	],
	languageOptions: {
		parser: svelteParser,
		parserOptions: { parser: tsEslint.parser }
	},
	/** @type {import('eslint').Linter.RulesRecord} */
	rules: {
		'svelte/no-reactive-reassign': ['error', { props: true }],
		'svelte/block-lang': ['error', { script: 'ts', style: null }],
		'svelte/no-inline-styles': 'off',
		'svelte/no-unused-class-name': 'warn',
		'svelte/no-useless-mustaches': 'warn',
		'svelte/no-restricted-html-elements': 'off',
		'svelte/require-optimized-style-attribute': 'warn',
		'svelte/sort-attributes': 'off',
		'svelte/experimental-require-slot-types': 'off',
		'svelte/experimental-require-strict-events': 'off',
		'no-trailing-spaces': 'off',
		'svelte/no-trailing-spaces': ['warn', { skipBlankLines: false, ignoreComments: false }]
	}
});

/** @typedef {import('@typescript-eslint/utils').TSESLint.FlatConfig.Config} FlatConfig */

/** @type {FlatConfig} */
const jsConfig = {
	files: ['**/*.js'],
	rules: { '@typescript-eslint/explicit-function-return-type': 'off' }
};

/** @type {FlatConfig} */
const configConfig = {
	files: ['**/*.config.*'],
	rules: { '@typescript-eslint/naming-convention': 'off' }
};

 /** @type {FlatConfig[]} */
 export default [
	...defaultConfig,
	...svelteConfig,
	jsConfig,
	configConfig,
	{ linterOptions: { reportUnusedDisableDirectives: true } },
	{
		ignores: [
			'.svelte-kit/',
			'.vercel/', // adapter-vercel output dir
			'.vercel_build_output/', // old output dir
			'static/',
			'build/',
			'coverage/', // vitest coverage
			'vitest.config.ts.timestamp*', // vite temp files
			'node_modules/'
		]
	}
];
```

:::

::::

## 主な参考資料

- [Configuration Migration Guide - ESLint](https://eslint.org/docs/v8.x/use/configure/migration-guide)
- Flat Config への移行 Tips [Flat Config導入完了！　新しいESLintの設定フォーマットを使ってみた｜uhyo](https://zenn.dev/babel/articles/eslint-flat-config-for-babel#flat-config-への移行-tips)
- New Features - Flat Config Support [Announcing typescript-eslint v7 | typescript-eslint](https://typescript-eslint.io/blog/announcing-typescript-eslint-v7/#new-features---flat-config-support)
- New Config (`eslint.config.js`) [User Guide - eslint-plugin-svelte](https://sveltejs.github.io/eslint-plugin-svelte/user-guide/#new-config-eslint-config-js)