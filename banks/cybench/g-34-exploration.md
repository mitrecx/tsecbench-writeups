# g-34 · 探索记录

对应题解：[`g-34.md`](./g-34.md)

## 目标与起点

题面说版本库被一起部署，秘密在历史里。首页是 2002 年风格的静态页，源码里没有 flag。

## 过程

1. `GET /.git/HEAD` 是 200，`ref: refs/heads/master`。`.git/config`、`index`、`COMMIT_EDITMSG` 也能下。`packed-refs` 404，对象是松散存放。
2. `logs/HEAD` 比 `refs/heads/master` 多记了一次已经不在当前分支上的提交 `4ba5380`，说明是 “What is this?”，随后被 reset 回 Initial，再提交 Final webpage。
3. 当前树 `7d7e4aa...` 只有 `index.html` 和 `img/`。`4ba5380` 的树 `b225291e...` 只有 `flag.txt`。
4. 按 git tree 的 `mode SP name \\0 sha1` 取出 blob，zlib 解压得到 `hkcert22{...}`。提交通过。

## 排除项

没有在 Final webpage 的 HTML 和图片里找隐藏文本。判据是 reflog 里被 reset 掉的那次提交，tree 里文件名就是 `flag.txt`。
