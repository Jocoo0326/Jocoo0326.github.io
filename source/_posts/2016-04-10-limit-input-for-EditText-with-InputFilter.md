---
layout: post
title: Limit input for EditText with InputFilter
date: 2016-04-10
categories:
- android
tags:
- EditText,
- InputFilter
---

项目中经常遇到EditText需要限制输入的情况，虽然EditText本身inputType自带了很多类型，如果遇到需要自定义输入限制时可以使用InputFilter接口，
<!-- more -->
例如需要限制输入表情符号，可以这样：

``` java

    public static class EmojiFilter implements InputFilter {

        @Override
        public CharSequence filter(CharSequence source, int start, int end, Spanned dest,
                                   int dstart, int dend) {
            if (source.length() >= 2) {
                if (containsEmoji(source.toString())) {
                    return "";
                }
            }
            return null;
        }
    }

    /**
     * 检测是否有emoji表情
     *
     * @param source
     * @return
     */
    public static boolean containsEmoji(String source) {
        int len = source.length();
        for (int i = 0; i < len; i++) {
            char codePoint = source.charAt(i);
            if (!isEmojiCharacter(codePoint)) { //如果不能匹配,则该字符是Emoji表情
                return true;
            }
        }
        return false;
    }

    /**
     * 判断是否是Emoji
     *
     * @param codePoint 比较的单个字符
     * @return
     */
    private static boolean isEmojiCharacter(char codePoint) {
        return (codePoint == 0x0) || (codePoint == 0x9) || (codePoint == 0xA) || (codePoint == 0xD) || ((codePoint >= 0x20) && (codePoint <= 0xD7FF)) || ((codePoint >= 0xE000) && (codePoint <= 0xFFFD)) || ((codePoint >= 0x10000));
    }

    protected void addInputFilter(InputFilter filter) {
        if (filter == null) {
            throw new IllegalArgumentException();
        }
        InputFilter[] filters = getFilters();
        InputFilter[] detFilters = new InputFilter[(filters == null ? 0 : filters.length) + 1];
        if (filters != null) {
            System.arraycopy(filters, 0, detFilters, 0, filters.length);
        }
        detFilters[detFilters.length - 1] = filter;
        setFilters(detFilters);
    }

```

然后使用addInputFilter添加到EditText中即可。

