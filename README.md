# Tailwind v4 `@source` and `.gitignore` Allow-List Conflict

> [!IMPORTANT]
> This issue has been fixed in Tailwind v4.3.1 see https://github.com/tailwindlabs/tailwindcss/issues/19844 for refrence

This repository is a minimal reproduction of a bug in Tailwind CSS v4 where the `@source` directive respects `.gitignore` patterns. When a project uses an allow-list `.gitignore` (starting with `*`) with explicit `!` exceptions, Tailwind cannot scan files in git-ignored directories, even when they are explicitly referenced via `@source`.

## The bug

Tailwind v4 uses git to determine which files to scan. If your `.gitignore` starts with `*` to deny everything and then uses `!` rules to allow specific paths, any directory not on the allow-list is invisible to Tailwind. Pointing `@source` at such a directory produces no output, and the CSS classes defined in those templates are silently missing from the build.

This is inconsistent with the deny-list approach. If instead the `.gitignore` explicitly lists `vendor/` as ignored (and allows everything else by default), Tailwind's `@source` works correctly and scans the directory as expected. The bug is specific to the allow-list pattern.

## Real-world context

This issue affects Adobe Commerce Cloud projects, which use the [magento-cloud `.gitignore`](https://github.com/magento/magento-cloud/blob/master/.gitignore). That file starts with `*` and uses allow-list exceptions, meaning `vendor/` is never explicitly allowed and is therefore invisible to Tailwind.

The standard [Magento 2 `.gitignore`](https://github.com/magento/magento2/blob/2.4-develop/.gitignore) uses the deny-list approach (`/vendor/*` is explicitly listed), and does not trigger this bug.

In a Hyvä theme, the Tailwind build scans both the child theme and the default theme in `vendor/hyva-themes/magento2-default-theme`, using patterns like:

```css
@import "tailwindcss" source(none);
@source "../../**/*.phtml";
@source "../../**/*.xml";
@source "../../../../../../vendor/hyva-themes/magento2-default-theme";
```

On Adobe Commerce Cloud projects, the `vendor/` path is silently skipped and styles from the default theme are missing from the build.

## Project structure

```
.gitignore                                          # allow-list (matches magento-cloud, vendor/ is not allowed)
app/
  design/frontend/theme/
    index.phtml                                     # uses bg-slate-500 (git-tracked)
    web/
      css/output.css                                # compiled output
      tailwind/
        input.css                                   # @source pointing to vendor/
        package.json
vendor/
  acme/theme/module/templates/
    component.phtml                                 # uses bg-blue-500 text-blue-50 (git-ignored)
```

## How to reproduce

```sh
cd app/design/frontend/theme/web/tailwind
npm install
npm run build
```

## Expected behavior

The compiled `output.css` should contain both `bg-slate-500` (from `index.phtml`) and `bg-blue-500`, `text-blue-50` (from `vendor/acme/theme/module/templates/component.phtml`), since both paths are explicitly listed via `@source`.

## Actual behavior

Only `bg-slate-500` appears in `output.css`. The classes from `vendor/` are missing because `vendor/` is git-ignored by the allow-list `.gitignore`, and Tailwind silently skips it despite the `@source` directive.

## Gitignore behavior contrast

**Allow-list (this repo, broken)** — [magento-cloud `.gitignore`](https://github.com/magento/magento-cloud/blob/master/.gitignore):

```gitignore
*
!/app
!/app/*/
!/app/design/**
```

`vendor/` is never explicitly allowed, so it is invisible to git and Tailwind. `@source` pointing to `vendor/` produces no output.

**Deny-list (works)** — [Magento 2 `.gitignore`](https://github.com/magento/magento2/blob/2.4-develop/.gitignore):

```gitignore
/vendor/*
/generated/
/pub/static/
```

`vendor/` is explicitly ignored, but git can still see it exists on disk. Tailwind's `@source` correctly scans it and includes the classes.

## Key files

**`input.css`** explicitly points `@source` at the vendor directory:

```css
@import "tailwindcss" source(none);
@source "../../**/*.phtml";
@source "../../**/*.xml";
@source "../../../../../../vendor/acme/theme";
```

Despite this, Tailwind does not scan `vendor/acme/theme` when the allow-list gitignore is in use.
