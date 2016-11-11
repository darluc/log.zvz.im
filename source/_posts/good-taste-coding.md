title: 如何拥有 Linus Torvalds 所说的编码“好品味”
date: 2016-11-11 13:57:43
tags:
- coding
---

在[最近一次对 Linus Torvalds （ Linux 之父 ） 的采访](https://www.ted.com/talks/linus_torvalds_the_mind_behind_linux)视频中，约 14 分 20 秒的位置，他提到了带着“好品味”编程的观点。好品味？采访者对其细节进行了追问，而 Linus 则是早有准备。

他首先展示了一个代码段。不过这不是一个有着“好品味”的代码。这个代码段是个坏样例，用作对比的原始代码。

```c
remove_list_entry(entry) 
{
  prev = NULL;
  walk = head;
  
  // Walk the list
  
  while (walk != entry) {
    prev = walk;
    walk = walk->next;
  }
  
  // Remove the entry by updating the
  // head or the previous entry
  
  if (!prev)
    head = entry->next;
  else
    prev->head = entry->next;
}
```
_无品味的代码示例_
<!--more-->

这是个 C 语言实现的函数，用于移除链表中的某个节点。共 10 行代码。

他让我们注意底部的 if 语句。就是这条 if 语句招致了他的批评。

我暂停了视频，研究了一下幻灯片。最近我有写过一些类似的代码。觉得 Linus 实际上就是在说我的品味差。我只好暂时放下我的自尊心，继续看视频。

Linus 向观众解释，从链表中移除一个节点时，有两种情况需要考虑。如果节点处于链表头，会于节点处于链表中的处理方式有所不同。而这就是那个 if 语句”坏品味“的来源。

不过既然他也承认这种特殊处理是必须的，那它到底坏在哪里？

接着，他又向观众展示了第二张幻灯片。这是他实现的同一个函数，不过是有着“好品味”的。

```c
remove_list_entry(entry)
{
  // The "indirect" pointer points to the
  // *address* of the thing we'll update
  indirect = &head;
  
  // Walk the list, looking for the thing that
  // points to the entry we want to remove
  
  while ((*indirect) != entry)
    indirect = &(*indirect)->next;
  
  // .. and just remove it
  *indirect = entry->next;
}
```

_好品味代码示例_

最初的 10 行代码现在缩减为了 4 行。

但是重点并不在于代码行数的减少。重点是那个 if 语句消失了，不再被需要。代码重构后， 无论节点在链表中的什么位置，都可以采用相同的处理过程来进行移除。

Linus 解释了这段新代码对于边界情况的消除才是重点。采访继续，进入了下个话题。

我研究了一会这个代码。Linus 是对的。第二个幻灯片所展示的代码更好。如果这是一道测试编码好坏的题目，那我可能已经挂了。消除条件判断的想法从未在我的脑子里出现过。而且我也像那坏例子一样写了不止一次了，因为我经常进行链表处理操作。

这个关于品味的展示，好处不仅仅在于教会你如何从一个链表中去除节点， 它还使你去思考你曾经写过的代码，你曾经在程序中写过的小小算法或许还有改进的空间，并且以一种你从未想过的方式出现。

所以最近我审阅项目代码时，这成为了我的关注点。很巧的是，这个项目也是使用的 C 语言。

就我的理解而言，“好品味”的关键是消除边界情况，而这些情况一般表现为条件判断语句。越少使用条件判断，你的代码“品味”就会更好。

下面有个我对代码进行改进的例子分享给大家。

## 初始化网格边缘

以下是我写的一个算法，用于初始化网格边缘的点，该网格使用一个多维数组表示：**grid\[rows\]\[cols\]** 。

这段代码的作用是初始化网格的边缘座标点 —— 即顶部一行，底部一行，左侧一列，和右侧一列。

为了完成工作，一开始我遍历了网格中的每个点，并且利用条件来判断他们是否处于边缘。代码如下：

```c
for (r = 0; r < GRID_SIZE; ++ r) {
  for (c = 0; c < GRID_SIZE; ++ c) {
    
    // Top Edge
    if (r == 0)
      grid[r][c] = 0;
    
    // Left Edge
    if (c == 0)
      grid[r][c] = 0;
    
    // Right Edge
    if (c == GRID_SIZE - 1)
      grid[r][c] = 0;
    
    // Bottom Edge
    if (r == GRID_SIZE - 1)
      grid[r][c] = 0;
  }
}
```

虽然这段代码可以正常工作，回头来看，确实有点问题：

1. 复杂度 —— 在二重循环中使用 4 个条件判断语句，看起来有些过于复杂了。
2. 效率 —— 如果 GRID_SIZE 等于 64，这个循环迭代了 4096 次，只是为了给 256 个边缘点赋值。

Linus 应该会觉得，这可真没品味。

所有我对它进行了一些修补。花了些时间后，我可以降低它的复杂度，使其只需要一个包含四个条件判断的 for 循环。这只是在复杂度上改进了一点点，却极大地提升了性能，因为它只循环了 256 次，每个边缘上的座标点对应一次循环。

```c
for (i = 0; i < GRID_SIZE * 4; ++ i) {
  
  // Top Edge
  if (i < GRID_SIZE)
    grid[0][i] = 0;
  
  // Right Edge
  else if (i < GRID_SIZE * 2)
    grid[i - GRID_SIZE][GRID_SIZE - 1] = 0;
  
  // Left Edge
  else if (i < GRID_SIZE * 3)
    grid[i - (GRID_SIZE * 2)][0] = 0;
  
  // Bottom Edge
  else
    grid[GRID_SIZE - 1][i - (GRID_SIZE * 3)] = 0;
}
```

这的确是一次改进。但是代码真是太丑陋了。而且并不易于理解。因此，我对这段代码并不满意。

所以我继续修改。这段代码是否能进一步优化呢？事实上，答案是肯定的。而且我最终形成的代码是令人吃惊得简单而优美，我都不敢相信，我居然花了这么长时间才发现这个办法。

下面就是代码的最终版本。它只包含了一个循环，而且没有条件判断。不但如此，循环也只需要 64 次。极大地改善了复杂度，并且提升了效率。

```c
for (i = 0; i < GRID_SIZE; ++i) {
  
  // Top Edge
  grid[0][i] = 0;
  
  // Bottom Edge
  grid[GRID_SIZE - 1][i] = 0;
  
  // Left Edge
  grid[i][0] = 0;
  
  // Right Edge
  grid[i][GRID_SIZE - 1] = 0;
}
```

这段代码每次循环都为四条不同的边缘进行初始化操作。它并不复杂，而且非常高效，又很容易理解。与最初版本相比，哪怕是与第二个版本相比，都称得上是天壤之别。

我表示非常满意。

那么，我就是一个有品味的程序员了么？

我认为我是，但并不是因为我上面完成的示例，或者其它别的什么原因...因为带着“好品味”编程比任何一段代码概念上都要广泛得多。Linus 也说他提供的例证太渺小了，并不足以将他的观点表达完全，描述清楚。

我认为 Linus 的意思是，拥有“好品味”的开发者，总是在开始编码前先概念化（ conceptualize ）他们将要构建的东西。他们定义工作中使用到的组件的边界，以及组件间彼此交互的方式。他们尽力确保所有东西都能彼此契合，且执行过程优雅。

这样做得到的结果，与 Linus  的“好品味”示例，或者我的示例会一样棒，只不过规模要大得多。

那么，在你的下个项目中，你想要如何展示出“好品味”呢？



> 翻译自：[Applying the Linus Torvalds "Good Taste" Coding Requirement](https://medium.com/@bartobri/applying-the-linus-tarvolds-good-taste-coding-requirement-99749f37684a#.y4pz33nn2)