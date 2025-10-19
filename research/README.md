## WHAT
利用AI 解决一些疑难问题，这些问题可能受限于个人的知识局限性，无法找到合理的解，但是现实中应该是有类似的解决方案的，因此我们提供一个方法，能够利用AI的`deep research`和`online search`功能，帮忙尽可能的发掘最佳的思路

## HOW
1个简单的prompt：
```
我现在遇到了一个问题，{{问题内容}}，请你帮我梳理金三年的相关文献 帮我整理出这些文献解决的问题及他们的创新点 帮我由浅入深的整理成一份调研报告
```

1个稍微复杂一些的prompt:
```
<background>
  {.....}
</background>
上面是我整理的一些Deep Research 类工具使用经验，，请参考这些经验（其中可能有些错误），帮我整理一份 Deep Research 类产品的使用经验手册，优先考虑 OpenAI Deep Research，同时也考虑 Grok Open Search，Gemini Deep Research等同类产品，重点分析：
- Deep Research 产品介绍
- Deep Research 提示词使用经验
- 注意事项
- 技术原理解析

检索时使用权威信息源，英文为主。

输出格式：
- 深度分析报告
- 中文报告
```
