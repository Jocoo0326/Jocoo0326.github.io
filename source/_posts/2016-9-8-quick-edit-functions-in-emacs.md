---
layout: post
title: Emacs中自定义快速编辑功能
date: 2016-09-08
categories:
- tools
tags:
- emacs lisp
---

emacs的一个强大之处就在于它不仅仅是一个编辑器，它还是一个emacs lisp解析器，可以通过emacs lisp扩展emacs的功能，下面是我自己扩展的用于提高编辑效率的emacs lisp函数
<!-- more -->

1. 在当前行的上方插入新行
``` lisp
;; insert new line above current line
(defun my/insert-new-line-before-current (times)
  (interactive "p")
  (move-beginning-of-line 1)
  (newline times)
  (previous-line times)
  (indent-for-tab-command))
(global-set-key (kbd "C-S-o") 'my/insert-new-line-before-current)
```

2. 在当前行的下方插入新行
``` lisp
;; insert new line below current line
(defun my/insert-new-line-below-current (times)
  (interactive "P")
  (move-end-of-line 1)
  (newline times)
  (indent-for-tab-command))
(global-set-key (kbd "C-o") 'my/insert-new-line-below-current)
```

3. 删除光标下的单词
``` lisp
;; delete word under cursor
(defun my/delete-word-under-cursor ()
  (interactive)
  (backward-word)
  (kill-word 1))
(global-set-key (kbd "C-c c i w") 'my/delete-word-under-cursor)
```

4. 复制单词
``` lisp
;; copy word
(defun my/copy-word (arg)
  (interactive "p")
  (let ((count (or arg 1)) (beg) (end))
    (if (= count 1)
	(let ((bnd (bounds-of-thing-at-point 'word)))
	  (setq beg (car bnd)
		end (cdr bnd)))
      (save-excursion
	(setq beg (point))
	(forward-word count)
	(setq end (point))))
    (copy-region-as-kill beg end)
    (message "word%s copied." (if (> count 1) "s" ""))))

(global-set-key (kbd "C-c w") 'my/copy-word)
```

5. 复制行
``` lisp
;; copy line
(defun my/copy-line (arg)
  (interactive "p")
  (let ((beg (line-beginning-position))
	(end (line-end-position arg)))
    (when mark-active
      (if (> (point) (mark))
	  (setq beg (save-excursion (goto-char (mark)) (line-beginning-position)))
	(setq end (save-excursion (goto-char (mark)) (line-end-position)))))
    (if (eq last-command 'my/copy-line)
	(kill-append (buffer-substring beg end) (< end beg))
      (kill-ring-save beg end)))
  (kill-append "\n" nil)
  (beginning-of-line (or (and arg (1+ arg)) 2))
  (if (and arg (not (= 1 arg))) (message "%d lines copied" arg)))
(global-set-key (kbd "C-c l") 'my/copy-line)
```

6. 查找光标下的文件
``` lisp
;; find file at position
(global-set-key (kbd "C-]") 'ffap)
```

根据需要后续保持更新.