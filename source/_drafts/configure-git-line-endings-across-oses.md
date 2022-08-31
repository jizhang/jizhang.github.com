---
title: Configure Git Line Endings Across OSes
tags: [git]
categories: Programming
---

In Linux, lines end with LF (Line Feed, `\n`), while in Windows, CRLF (Carriage Return + Line Feed, `\r\n`). When developers using different operating systems contribute to the same Git project, line endings must be handled correctly, or `diff` and `merge` may break unexpectedly. Git provides several solutions to this problem, including configuration options and file attributes.

## TL;DR

### Approach 1

Set `core.autocrlf` to `input` in **Windows**. Leave Linux/macOS unchanged.

```
git config --global core.autocrlf input
```

### Approach 2

Create `.gitattributes` under the project root, and add the following line:

```
* text=auto eol=lf
```

<!-- more -->

## Use consistent line endings

I suggest using LR in all OSes. Modern editors are capable of recoganizing and handling line endings across platforms. Even [Notepad in Windows 10][1] can display text files with LRs correctly. Usually we have an [`.editorconfig`][2] file in the project, so that various editors with plugin installed will behave the same when handling line endings, as well as charset and indent.

```
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
max_line_length = 100
```

This consistency also lies in Git itself. When you enable Git to handle line endings for you, either by `core.autocrlf` or `.gitattributes`, Git always stores LFs in the repository.

## The classic `core.autocrlf` option

[`core.autocrlf`][3] has three options:

* `false` The default value, meaning Git will not touch the files when checking in or out of the repository. Check-in means committing files to the repository; check-out means writing to the working directory.
* `true` Git will convert LR to CRLF when checking out of the repository, and convert them back to LF when checking in.
* `input` Git checks out the files *as-is*, and converts CRLF to LF when checking in.

When `core.autocrlf` is set to `input`, Git will give you a warning when adding text files with CRLF endings:

```
warning: CRLF will be replaced by LF in test.txt.
```

Only text files will be processed by Git, but sometimes Git may mistakenly treat binary files as text files and corrupt the data by replacing CRLF with LF. So Git provides a `core.safecrlf` option that checks if it can convert LF back to CRLF and produce the exact same file content. If it is not the case, Git rejects this operation with an error:

```
fatal: LF would be replaced by CRLF in test.bin
```

This setting also causes problem when you have a mixture of LF and CRLF in one file, because Git will detect that it cannot reproduce the original file when checking out. In this case, line endings need to be fixed manually.

## Configure end-of-line in Git attributes

There are two caveats in the `core.autocrlf` approach. First, it is a configuration that needs to be set manually by every developer, either globally or locally. Second, it may corrupt binary files. So newer version of Git provides the attribute mechanism, that saves configurations into a file named `.gitattributes`, and just like `.editorconfig`, this file should be checked into the repository so that all developers may share the same config. Git attributes also support path wildcards, so users can specify which files should be processed as text files. For instance:

```
# Auto detect file types, if no further configs are given. Set end-of-line to LF.
* text=auto eol=lf

# Specify the following file types to be text, and do the CRLF/LF conversion.
*.py text eol=lf
*.ts text eol=lf

# Leave the binary files as-is.
*.png binary
```

`binary` is a macro for `-text -diff`, meaning Git will *not* process this file as text files or generate diffs in `git diff`. Git attributes take precedence over the `core.autocrlf` config, and will fall back to it when file does not match the wildcards.

Another related config is `core.eol`, which only takes effect if a file has the `text` attribute. Consider it as the default value for `eol` in `.gitattributes`, but obviouly it should not be used since it is also a config that needs to be set manually.

More details on Git attributes can be found in the [official document][4].

## Renormalize after setting up end-of-line

For existing projects, there is a command that normalizes line endings for all files.

```
git add --renormalize .
git commit -m 'Normalize line endings.'
```

## References

* https://stackoverflow.com/a/2354278/1030720
* https://adaptivepatchwork.com/2012/03/01/mind-the-end-of-your-line/
* https://docs.github.com/en/get-started/getting-started-with-git/configuring-git-to-handle-line-endings


[1]: https://devblogs.microsoft.com/commandline/extended-eol-in-notepad/
[2]: https://editorconfig.org/
[3]: https://git-scm.com/docs/git-config#Documentation/git-config.txt-coreautocrlf
[4]: https://git-scm.com/docs/gitattributes#_text
