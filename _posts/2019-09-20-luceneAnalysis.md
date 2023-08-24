---
layout: post
title: lucene词条分析
tags:
- lucene
categories: lucene
description: analysis
---

lucene analysis 就是一个解决如何将一个长句解析成多个词条的工具包。

# 核心类图

<img src="/assets/img/lucene/analysisAnalysis.svg" />

- CharTermAttribute, 记录token的词条文本信息。
- OffsetAttribute, 记录token的偏移量位置信息。
- PositionLengthAttribute, 记录token的占位宽度信息，默认值为1，只在图结构(graph)中作用较大。
- PositionIncrementAttribute, 记录token相对于上一个token的位置增长量信息，默认值为1，在多词干(stem)单词搜索和忽略停止词短语匹配的
  场景下有发挥作用。
 


# 使用TokenStream API

- 创建TokenStream对象。
- 调用stream的reset方法。
- 检索stream内部所有的Attribute。
- 一直调用incrementToken直到返回false。
- 调用stream的end方法。
- 调用stream的close方法。
  
```java
public class MyAnalyzer extends Analyzer {

    @Override
    protected TokenStreamComponents createComponents(String fieldName) {
        WhitespaceTokenizer whitespaceTokenizer = new WhitespaceTokenizer(); // 1
        return new TokenStreamComponents(whitespaceTokenizer); // 内部会调用 whitespaceTokenizer.setReader方法
    }

    public static void main(String[] args) throws IOException {
        // text to tokenize
        final String text = " This is a demo of the TokenStream 😊 API";

        MyAnalyzer analyzer = new MyAnalyzer();
        // tokenStream会回调上面的createComponents方法
        // 这里返回的stream就是上面标记1处的WhitespaceTokenizer
        TokenStream stream = analyzer.tokenStream("field", new StringReader(text));

        // get the CharTermAttribute from the TokenStream
        CharTermAttribute termAtt = stream.addAttribute(CharTermAttribute.class);

        try {
            stream.reset();

            // print all tokens until stream is exhausted
            while (stream.incrementToken()) {
                // This
                //is
                //a
                //demo
                //of
                //the
                //TokenStream
                //😊
                //API
                System.out.println(termAtt.toString());
            }

            stream.end();
        } finally {
            stream.close();
        }
    }
}
```
源码见[这里](https://github.com/gskeno/lucene/blob/branch_9_7_notes/lucene/analysis/common/src/test/org/apache/lucene/analysis/MyAnalyzer.java)

Analyzer负责创建分词组件`Tokenizer` 和 `TokenFilter`，Tokenizer主要用来对原始句子进行分词获得token，TokenFilter的实现使用了`包装者设计模式`，主要用来对token进行过滤或更改，二者都继承自`TokenStream`。

## token长度过滤
```java
public class MyLengthAnalyzer extends Analyzer {
    @Override
    protected TokenStreamComponents createComponents(String fieldName) {
        WhitespaceTokenizer whitespaceTokenizer = new WhitespaceTokenizer();
        // 包装者设计模式
        // 会过滤掉长度<3的token
        LengthFilter lengthFilter = new LengthFilter(whitespaceTokenizer, 3, Integer.MAX_VALUE);
        TokenStreamComponents tokenStreamComponents = new TokenStreamComponents(whitespaceTokenizer, lengthFilter);
        return tokenStreamComponents;
    }

    public static void main(String[] args) throws IOException {
        // text to tokenize, 长度小于3的词条不会被输出
        final String text = "This is a demo of the TokenStream API";

        MyLengthAnalyzer analyzer = new MyLengthAnalyzer();
        TokenStream stream = analyzer.tokenStream("field", new StringReader(text));

        // get the CharTermAttribute from the TokenStream
        CharTermAttribute termAtt = stream.addAttribute(CharTermAttribute.class);

        try {
            stream.reset();

            // print all tokens until stream is exhausted 过滤掉长度小于3的token
            // This
            // demo
            // the
            // TokenStream
            // API
            while (stream.incrementToken()) {
                System.out.println(termAtt.toString());
            }

            stream.end();
        } finally {
            stream.close();
        }
    }
}
```

## 自定义Attribute
对于每个token而言，我们假设其首字母为大写时是名词，其他情况为"Unknown"，要在分词器中使用，需要以下几步。

- 自定义Attribute子接口，并编写实现子类。
- 编写一个TokenFilter过滤器，对每一个token进行鉴别其词性(是否是名词)
  
```java
public interface PartOfSpeechAttribute extends Attribute {
    enum PartOfSpeech {
        Noun, Verb, Adjective, Adverb, Pronoun, Preposition, Conjunction, Article, Unknown
    }

    public void setPartOfSpeech(PartOfSpeech pos);

    public PartOfSpeech getPartOfSpeech();
}

public final class PartOfSpeechAttributeImpl extends AttributeImpl
        implements PartOfSpeechAttribute {

    private PartOfSpeech pos = PartOfSpeech.Unknown;

    public void setPartOfSpeech(PartOfSpeech pos) {
        this.pos = pos;
    }

    public PartOfSpeech getPartOfSpeech() {
        return pos;
    }

    @Override
    public void clear() {
        pos = PartOfSpeech.Unknown;
    }

    @Override
    public void reflectWith(AttributeReflector reflector) {

    }

    @Override
    public void copyTo(AttributeImpl target) {
        ((PartOfSpeechAttribute) target).setPartOfSpeech(pos);
    }

}


class PartOfSpeechTaggingFilter extends TokenFilter {
    PartOfSpeechAttribute posAtt = addAttribute(PartOfSpeechAttribute.class);
    CharTermAttribute termAtt = addAttribute(CharTermAttribute.class);

    protected PartOfSpeechTaggingFilter(TokenStream input) {
        super(input);
    }

    public boolean incrementToken() throws IOException {
        if (!input.incrementToken()) {return false;}
        posAtt.setPartOfSpeech(determinePOS(termAtt.buffer(), 0, termAtt.length()));
        return true;
    }

    // determine the part of speech for the given term
    protected PartOfSpeechAttribute.PartOfSpeech determinePOS(char[] term, int offset, int length) {
        // naive implementation that tags every uppercased word as noun
        if (length > 0 && Character.isUpperCase(term[0])) {
            return PartOfSpeechAttribute.PartOfSpeech.Noun;
        }
        return PartOfSpeechAttribute.PartOfSpeech.Unknown;
    }
}

public class PartOfSpeechAnalyzer extends Analyzer {

    @Override
    protected TokenStreamComponents createComponents(String fieldName) {
        final Tokenizer source = new WhitespaceTokenizer();
        TokenStream result = new LengthFilter( source, 3, Integer.MAX_VALUE);
        result = new PartOfSpeechTaggingFilter(result);
        return new TokenStreamComponents(source, result);
    }

    public static void main(String[] args) throws IOException {
        // text to tokenize
        final String text = "This is a demo of the TokenStream API";

        PartOfSpeechAnalyzer analyzer = new PartOfSpeechAnalyzer();
        TokenStream stream = analyzer.tokenStream("field", new StringReader(text));

        // get the CharTermAttribute from the TokenStream
        CharTermAttribute termAtt = stream.addAttribute(CharTermAttribute.class);

        // get the PartOfSpeechAttribute from the TokenStream
        PartOfSpeechAttribute posAtt = stream.addAttribute(PartOfSpeechAttribute.class);

        try {
            stream.reset();

            // print all tokens until stream is exhausted
            while (stream.incrementToken()) {
                System.out.println(termAtt.toString() + ": " + posAtt.getPartOfSpeech());
            }

            stream.end();
        } finally {
            stream.close();
        }
    }
}

```
输出结果
```text
This: Noun
demo: Unknown
the: Unknown
TokenStream: Noun
API: Noun
```
源码在[这里](https://github.com/gskeno/lucene/blob/branch_9_7_notes/lucene/analysis/common/src/test/org/apache/lucene/analysis/miscellaneous/PartOfSpeechAnalyzer.java)

# 参考
- [lucene 9.7 analysis](https://lucene.apache.org/core/9_7_0/core/org/apache/lucene/analysis/package-summary.html)