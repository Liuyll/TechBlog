## GEBI or QS

#### 一般区别：

1. GEBI 返回 HTML Collection，这是动态数组，随着HTML变化而变化
2. QS 返回静态的 Node List 它不随HTML变化而变化

#### 应用场景

1. QS性能远慢于动态的GEBI
2. QS只应用于 <b>需要某个特定时间的DOM集合</b>时

## append or appendChild

1. appendChild不支持文本节点，而append支持

   `appendChild(document.createTextElement('qweqew'))`

   `append('qweqwe')`

2. append支持添加多个元素

3. append没有返回值