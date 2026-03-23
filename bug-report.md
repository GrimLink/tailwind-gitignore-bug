# `@source` silently ignores paths excluded by an allow-list `.gitignore`

**What version of Tailwind CSS are you using?**

v4.1.13

**What build tool (or framework if it abstracts the build tool) are you using?**

@tailwindcss/cli 4.1.13

**What version of Node.js are you using?**

v25.6.1

**What browser are you using?**

N/A

**What operating system are you using?**

macOS 26.3.1

**Reproduction URL**

https://github.com/grimlink/tailwind-gitignore-bug

**Describe your issue**

When a project uses an allow-list `.gitignore` pattern (starting with `*` to exclude everything, followed by `!` rules to allow specific paths), Tailwind v4 silently ignores `@source` directives that point to git-ignored directories. No error or warning is produced, and the CSS classes from those templates are missing from the compiled output.

This affects Adobe Commerce Cloud projects, which use the [magento-cloud `.gitignore`](https://github.com/magento/magento-cloud/blob/master/.gitignore). That file starts with `*` and selectively allows only application code and config, leaving `vendor/` invisible to git and Tailwind.

The behavior is inconsistent depending on which gitignore style is used:

- **Allow-list** (`*` first, then `!` exceptions): `@source` pointing to `vendor/` fails silently. No classes are generated.
- **Deny-list** (`vendor/` explicitly listed as ignored, as in the [Magento 2 `.gitignore`](https://github.com/magento/magento2/blob/2.4-develop/.gitignore)): `@source` pointing to `vendor/` works correctly. Classes are generated as expected.

The same `@source` directive, the same `vendor/` directory on disk, two different outcomes based solely on how the `.gitignore` is structured.

**Steps to reproduce**

1. Create a `.gitignore` with an allow-list pattern:
   ```
   *
   !/app
   !/app/design/**
   ```
2. Place a template with Tailwind classes inside `vendor/` (git-ignored):
   ```html
   <!-- vendor/acme/theme/module/templates/component.phtml -->
   <div class="bg-blue-500 text-blue-50">This should be blue</div>
   ```
3. Reference the vendor directory via `@source` in your CSS entry point:
   ```css
   @import "tailwindcss" source(none);
   @source "../../**/*.phtml";
   @source "../../**/*.xml";
   @source "../../../../../../vendor/acme/theme";
   ```
4. Run the Tailwind build.

**Expected behavior**

`bg-blue-500` and `text-blue-50` are present in the compiled CSS, because the path is explicitly declared via `@source`.

**Actual behavior**

`bg-blue-500` and `text-blue-50` are absent from the compiled CSS. Tailwind does not scan the `vendor/` directory because git considers it non-existent (due to the allow-list gitignore). No warning or error is emitted.

Only classes from git-tracked files (e.g., `bg-slate-500` from a file under `app/design/`) appear in the output.
