---
layout: post
title: Emacs lisp使用宏扩展编辑功能
date: 2016-11-08
categories:
- tools
tags:
- Emacs
---
Lisp的宏是一个非常神奇的特性，可以在编译时动态运行代码，也就是动态生成代码的能力
<!-- more -->

我们都知道编程中相同的数字要提炼成常量，相同的调用要提炼成方法，那类似的方法定义呢？C/C++中有宏和模板，但只是简单的字符串替换，而lisp中可以定义成宏；

Emacs lisp code snippets：
``` lisp
;;------------------------------------------------------------------------------------------
;; copy/delete chars words lines paragraphs
;;------------------------------------------------------------------------------------------

;; operate region macro
(defmacro jocoo/region-operate (op-name unit op)
  `(defun ,(intern (concat "jocoo/" op-name "-" unit "-under")) (arg)
     (interactive "p")
     (let ((count (or arg 1)) (beg) (end) (bound))
       (setq bound (bounds-of-thing-at-point (quote ,(intern unit))))
       (setq beg (car bound))
       (save-excursion
	 (goto-char beg)
	 (,(intern (concat "forward-" unit)) count)
	 (setq end (point)))
       (,op beg end)
       (message ,(concat op-name " " unit "%s") (if (> count 1) "s" "")))))


;; char operation
(my/region-operate "copy" "char" copy-region-as-kill)
(my/region-operate "delete" "char" kill-region)
(global-set-key (kbd "C-c c c") 'my/copy-char-under)
(global-set-key (kbd "C-c d c") 'my/delete-char-under)

;; word operation
(my/region-operate "copy" "word" copy-region-as-kill)
(my/region-operate "delete" "word" kill-region)
(global-set-key (kbd "C-c c w") 'my/copy-word-under)
(global-set-key (kbd "C-c d w") 'my/delete-word-under)

;; line operation
(my/region-operate "copy" "line" copy-region-as-kill)
(my/region-operate "delete" "line" kill-region)
(global-set-key (kbd "C-c c l") 'my/copy-line-under)
(global-set-key (kbd "C-c d l") 'my/delete-line-under)

;; paragraph operation
(my/region-operate "copy" "paragraph" copy-region-as-kill)
(my/region-operate "delete" "paragraph" kill-region)
(global-set-key (kbd "C-c c p") 'my/copy-paragraph-under)
(global-set-key (kbd "C-c d p") 'my/delete-paragraph-under)
```
一个宏的8次展开，生成了8个函数的定义，代码量指数级减少；

### 思考
```
All problems in computer science can be solved by another level of indirection.
by David Wheeler
```
Lisp的宏可以看作是抽象了函数的定义，有了代码生成代码的能力；
JVM抽象了机器指令，有了跨平台的能力；
C抽象了汇编语言，有了简洁优雅的编程语言；
CMAKE抽象了工具链，有了统一的构建脚本；
设计模式抽象了解决问题的经验法则，有了程序设计的准则；
移动互联网抽象了机器的计算能力；减少了沟通成本；
etc.
技术的发展在于解放于人
