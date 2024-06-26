title: 大型语言模型的历史、时间线和未来
date: 2024-4-01 16:05:27
tags:

- ai
- llm

---

### 引言

人工智能的发展速度之快前所未有，尤其是ChatGPT频频成为头条新闻，大型语言模型（LLMs）的戏剧性演变也不断出现在媒体圈中。全球数百万人迅速采用了对话式人工智能工具作为他们日常生活的一部分。这些工具不仅因其惊人的能力、效率令人着迷，同时也因不加以良好监管则可能带来的潜在危险而令人恐惧。

那么，一切从何开始，又将走向何方？这是我们将尝试进一步深入探讨的百万美元问题。为了帮助您更好地理解这个快速发展的领域，在本文中，我们涵盖了以下主题：

- 简要介绍LLMs的历史。

- 变换器和ChatGPT的崛起。

- 训练LLMs。

- LLMs的类型和应用。

- 当前的限制、挑战和潜在危险。

- 这些模型的未来前景。从学习过程到人工反馈的强化学习，让我们更深入的探究语言模型的基础，它们是如何被训练的以及它们是如何工作的。

<!--more-->

### 什么是大型语言模型？

大型语言模型的核心是一种机器学习模型，它能够通过深度神经网络理解和生成人类语言。语言模型的主要任务是计算在给定输入后，一个词在句子中出现的概率：例如，“天空是______”，最可能的答案是“蓝色”。模型能够在给定大量文本数据集（或语料库）后预测句子中的下一个词。基本上，它在学习识别单词组合的各种不同模式。通过这个过程，你可以得到一个预训练的语言模型。

经过一些微调，这些模型可以产生多种实际用途，如翻译，或在特定知识领域（如法律或医学）建立专业知识。这个过程被称为迁移学习，它使得模型能够将从一个任务中获得的知识应用到另一个任务上。

语言模型的“大型”是指其架构的规模。这反过来基于神经网络的人工智能，这就像人类大脑中的神经元一起作用来进行学习和处理信息。此外，LLMs包含大量参数（例如，GPT有超过100亿个），通过自监督或半监督学习在大量未标记的文本数据上进行训练。通过前者，模型能够从未注释的文本中学习，考虑到依赖手动标记数据的成本高昂，这是一个巨大的优势。

此外，参数更多的大型网络相较与较小的网络在保留信息和识别模式方面表现出了更好的性能和更大的容量。模型越大，在训练过程中能够学习的相关信息就越多，这反过来又使其预测更加准确。虽然这在传统意义上可能是正确的，但有一个需要注意的问题：人工智能公司和开发者们都在寻找应对训练大型语言模型所需的过度计算成本和能源挑战的方法，通过引入更小、训练更优化的模型来实现这一点。

尽管大型语言模型主要被训练在简单任务上，比如预测句子中的下一个单词，但令人惊讶的是它们能够捕捉到语言结构和意义的多少，更不用说它们能够掌握的大量事实了。

### 关键术语

这是一个相对较新的领域，有很多术语需要你去熟悉。以下是可以帮助你更好地理解 LLM 世界的主要术语：

- **注意力** — 评估每个通过LLM传递的标记影响的统计装置。
- **嵌入** — 捕捉单词含义及其在上下文中的关系的数值表示。
- **变换器** — 构成大多数LLMs基础的神经网络架构。
- **提示** — 用户提供给LLM以引出响应或执行任务的输入。
- **指令调整** — 训练语言模型回答不同的提示以学习如何回答新的提示。
- **RLHF（人类反馈强化学习）** — 根据人类偏好调整模型的技术。

### 大型语言模型的简史：最初是Eliza...

当古人小心翼翼地将他们的知识记录在莎草纸上，并存放在传说中的亚历山大图书馆时，他们无法梦想到，千年后他们的后代能够在指尖上获得所有这些知识以及更多。这就是大型语言模型的力量和美丽。LLMs不仅能回答问题和解决复杂问题，还能总结大量作品，以及从各种语言中翻译和推导上下文。

大型语言模型的基础可以追溯到20世纪50年代进行的神经网络和神经信息处理系统的实验，这些实验允许计算机处理自然语言。IBM和乔治敦大学的研究人员合作创建了一个能够自动将短语从俄语翻译成英语的系统。作为机器翻译的一个显著示范，从此该领域的研究开始起步。

LLMs的想法首次出现在20世纪60年代的Eliza中：这是世界上第一个聊天机器人，由麻省理工学院的研究员Joseph Weizenbaum设计。Eliza标志着自然语言处理（NLP）研究的开始，为未来的更复杂的LLMs奠定了基础。

然后在约30年后的1997年，长短期记忆（LSTM）网络出现了。它们的出现导致了更深层次、更复杂的神经网络，能够处理更多的数据。斯坦福大学的CoreNLP套件在2010年推出，是发展的下一个阶段，它允许开发者进行情感分析和已命名实体的识别。

随后，在2011年，Google Brain的一个具有如词嵌入等高级特性的较小版本出现了，这使得NLP系统能够更清晰地理解上下文。2017年变换器模型的出现是一个重大的转折点。GPT代表生成预训练变换器，能够生成或“解码”新文本。另一个例子是BERT - 来自变换器的双向编码器表示。BERT可以根据编码器组件预测或分类输入文本。

从2018年开始，研究人员专注于构建越来越大的模型。正是在2019年，谷歌的研究人员推出了BERT，这是一个双向的、3.4亿参数的模型（同类中第三大的模型），能够确定上下文，使其能够适应各种任务。通过在大量未结构化数据上通过自监督学习预训练BERT，模型能够理解单词之间的关系。不久，BERT成为了自然语言处理任务的首选工具。事实上，正是BERT在Google搜索中处理了每一个基于英语的查询。

### 变换器和ChatGPT的崛起

在BERT变得更加精细的同时，OpenAI拥有15亿参数的GPT-2，成功地产生了令人信服的文本生成能力。然后，在2020年，他们发布了拥有1750亿参数的GPT-3，这为LLMs设定了标准，并成为了ChatGPT的基础。正是在2022年11月ChatGPT的发布，普通公众开始真正注意到LLMs的影响。即使是非技术用户也可以提示LLM，接收快速响应，并进行对话，引起了让人既兴奋又忧虑的轰动。

主要地，是带有编码器-解码器架构的变换器模型催生了更大更复杂的LLMs的创造，成为Open AI的GPT-3、ChatGPT等的催化剂。利用两个关键组件：词嵌入（使模型能够在上下文中理解单词）和注意力机制（使模型能够评估单词或短语的重要性），变换器在确定上下文方面特别有帮助。由于它们能够一次性处理大量数据，自那时起它们一直在推动该领域的变革。

最近，OpenAI推出了GPT-4，估计有一万万亿参数——是GPT-3的五倍，大约是BERT首次发布时的3000倍。我们将在后面更详细地介绍GPT系列的演变。

更加直观地，这是从Eliza到OpenAI的GPT的LLMs发展时间线的简单概述：
![history of large language models](https://pic.zvz.im/blogimg/20240403115340.png)

### 深入了解 GPT LLMs 的演进历程

GPT-1通过执行简单任务（如回答问题）启动了LLMs的演变。当GPT-2发布时，模型已经显著增长，参数增加了10倍以上。GPT-2能产生类似人写的文本，并自动执行某些任务。随着GPT-3的引入，公众能够访问这项创新技术。GPT-3将问题解决引入了这个组合中。GPT-3.5扩大了系统的能力，变得更加流畅且成本更低。

到目前为止，最新的变体是GPT-4，它具有显著的增强功能，如使用计算机视觉解释视觉数据的能力（与使用GPT-3.5的ChatGPT不同）。GPT-4接受文本和图像作为输入。GPT-4能够通过多项标准化考试，这对于学生来说是个好消息，相反对于教育者来说却是个坏消息。不仅如此，它还在学习幽默。通过解释计算机视觉，这项新技术正在用学习和使用人类语言类似的方式来学习其它形式的幽默。准备好迎接AI生成的笑话吧！

更重要的是，“可操纵性”是最新的进步，它允许GPT用户定制其输出的结构以满足他们的特定需求。基本上，可操纵性指的是控制或修改语言模型行为的能力——这涉及到使LLM采用不同的角色，遵循用户指令，或以特定的语调说话。可操纵性允许用户随意改变LLM的行为，并命令它以不同的风格或声音写作。可能性是无限的！

### 越大越好？

正如我们之前简要提到的，研究表明，具有更多参数的较大语言模型编程语言具有更好的性能，从而促使开发人员和AI研究人员之间展开了一场竞赛，以创建更大更好的LLMs——参数越多，越好！

最初的GPT只有几百万个参数，变成了像BERT和GPT-2这样的模型，包含数亿个参数。最近的示例包括GPT-3，它拥有1750亿个参数，而Megatron-Turing的语言模型已经超过了5000亿个参数（在最后四年的每三个半月中大约是两倍）！

然而，AI实验室DeepMind的RETRO（Retrieval-Enhanced Transformer）模型已经证明，它可以胜过其他比它大25倍的现有模型。这是对训练更大模型的显著缺点的一个优雅解决方案，因为它们通常需要更多的时间、资源、金钱和计算能量来训练。不仅如此，RETRO已经证明其大型模型有潜力通过增强的过滤能力减少有害和有毒信息。所以，更大并不总是更好！下一个巨大的LLM的训练过程可能需要所有可用的文本和训练数据，而更小模型和更优化训练方式可能是解决方案。

此外，在GPT-4发布后不久，OpenAI的首席执行官Sam Altman表示，他认为巨型模型的时代已经达到了不可逆转的点，因为数据中心和网上的信息数量有限。许多研究人员现在同意，通过定制，更小的LLMs可以和大型模型一样有效，甚至更有效。

### 训练大型语言模型

LLMs通过自监督学习在大量未结构化数据上进行训练。在这个过程中，模型接受长单词序列，其中有一个或多个单词缺失。用户向LLM提供提示（模型用其作为起点的文本片段）。最初，模型将提示中的每个标记转换为其嵌入。然后，它使用这些嵌入来评估所有可能的标记跟随的概率。下一个标记是基于部分随机性选择的，并且重复该过程，直到模型选择一个停止标记。

那么，你需要多少数据来训练一个LLM？一个好的经验法则是将参数和训练数据集翻倍。根据当前的研究，大多数LLM实际上是欠训练的。DeepMind进行了一项研究，以确定训练变换器语言模型的最佳模型、参数大小和所需标记数量。该团队在50亿到5000亿标记上训练了超过400个语言模型，从7000万到160亿参数。他们发现，对于计算最优的训练，模型大小和标记数量应该是相等的。

### 训练计算最优的大型语言模型

你可能已经听说过著名的Chinchilla案例研究。我们在另一篇帖子中更详细地介绍了它[（大型语言模型训练101）](https://toloka.ai/blog/training-large-language-models-101/)。文章主要介绍了该团队研究了三种确定模型大小和训练标记数量之间关系的方法。所有三种方法都指向了这样一个想法：相对平等地增加模型的大小和训练标记数量将实现更好的性能。

他们通过训练一个名为Chinchilla的模型来测试他们的假设，该模型与其更大的模型等效Gopher具有相同的计算预算，但参数更少，数据是其四倍。他们发现，更小、更优化训练的模型表现出更好的性能：他们的计算最优的70亿模型Chinchilla在1.4万亿标记上训练，性能超过了Gopher（一个280亿参数模型），同时显著降低了推理成本。

有趣的是，DeepMind团队发现，一个7.5亿参数的RETRO模型在16个数据集中的9个上胜过Gopher。

### 大型语言模型的类型

LLMs可以分解为三种类型：预训练模型、微调模型和多模态模型。取决于各自的目标，每种类型都有其自己的优势：

- **预训练模型** 在大量数据上进行训练，这有助于它们理解广泛的语言模式和结构。一个优点是，预训练模型往往在语法上是正确的！
- **微调模型** 在大型数据集上进行预训练，然后在较小的数据集上针对特定任务进行微调。它们特别适合情感分析、回答问题和分类文本。
- **多模态模型** 结合文本与其他模式，如图像或视频，以创建更高级的语言模型。它们可以生成图像的文本描述，反之亦然。

### 大型语言模型的应用

LLMs有多种应用，包括总结不同文本、构建更有效的数字搜索工具和作为聊天机器人。我们将更进一步地探究四个关键应用：

### 1. 进化的对话式AI

LLMs已经证明能够在对话中生成相关和连贯的回应。想想聊天机器人和虚拟助手。类似地，LLMs正在帮助使语音识别系统更准确。请关注由于这一点而即将出现的所有新应用程序！

### 2. 理解文本中的情感

LLMs擅长情感分析和提取主观信息，如情感和观点。应用包括客户反馈、社交媒体分析以及品牌监控。

### 3. 高效的机器翻译

得益于LLMs，翻译系统变得更加高效和准确。通过打破语言障碍，LLMs正在使全球人类分享知识和相互沟通成为可能。

### 4. 文本内容创作

更令人惊讶的是，LLMs可以生成各种形式的文本，如新闻文章和产品描述。它们甚至已经在创意写作方面取得了显著的成功——再次让许多学生在提交学期论文时感到高兴（以及许多老师感到头疼）！

### 当前的限制和挑战

伴随着AI、机器学习模型以及整体LLMs的惊人进步，仍然有许多挑战需要克服。错误信息、恶意软件、歧视性内容、抄袭和不真实的信息可能导致意外或危险的后果；这些问题对这些模型提出了质疑。

更重要的是，当偏见不经意间被引入到基于LLM的产品（如GPT-4）中时，它们可能在某些主题上显得“确定但错误”。这有点像当你听到一个政治家谈论他们一无所知的事情时的情形。克服这些限制是建立公众对这项新技术信任的关键。这就是RLHF发挥作用的地方，以帮助控制或引导大规模AI系统。

以下是需要解决的一些主要问题：

### 伦理和隐私问题

目前，没有多少法律或限制规定LLMs的使用，由于大型数据集包含大量机密或敏感数据（不仅限于个人数据盗窃、版权和知识产权侵权等），这引发了伦理、隐私甚至用户的心理问题（当他们在AI生成的论坛中寻求答案时）。

### 偏见和成见

由于LLMs的训练是基于不同的信息来源的，它们可能会无意中重复这些来源中的偏见。包括文化、种族、性别等在内的偏见都可能成为问题。这些偏见可能对现实生活中产生影响，如招聘决策、医疗护理或财务结果。

### 环境影响和计算成本

训练大型语言模型需要大量的计算能力，这影响能源消耗和碳排放。不仅如此，它还很贵！许多公司，特别是较小的公司，根本负担不起。

为了对抗这些潜在的危害，研究人员正专注于围绕三个主要支柱设计LLMs：有用、真实和无害。如果一个LLM能够坚持所有三个原则，它就被认为是“一致的”——这是一个具有主观性的术语。RLHF在这种情况下提供了一个有用的解决方案。

### 从人类反馈中进行强化学习

RLHF有能力解决上面列出的所有挑战，这引发了一个问题：机器能学习人类价值观吗？

在基本层面上，RLHF利用人类反馈来生成人类偏好数据集，以保证奖励函数能得到期望的结果。人类反馈可以通过多种方式获得：

- **优先级顺序**：人们按偏好顺序对输出进行排名。
- **示范操作**：人类为提示编写首选答案。
- **更正**：人类编辑模型的输出以纠正负面行为。
- **自然语言输入**：人类用语言提供输出的描述或批评。一旦创建了奖励模型，它就用于使用强化学习训练基线模型，该模型利用奖励模型构建人类价值策略，然后语言模型使用该策略生成响应。该如何使用RLHF产生更好、更安全、更具吸引力的响应，ChatGPT给出了一个很好的示范。

RLHF是语言模型领域的一项重大进步，提供了更受控和可靠的用户体验。但其中有一个代价：RLHF引入了用于训练奖励模型的偏好数据集中的贡献者的偏见。因此，尽管ChatGPT旨在提供有用、诚实和安全的答案，但它仍然受到标注者对这些答案的解释的影响。虽然RLHF提高了一致性（这对于LLMs在搜索引擎中的使用是很有益的），但它以牺牲创意和思想多样性为代价。在这个领域仍然有很多未知的东西。

### LLMs的未来展望

虽然ChatGPT是最新的闪亮事物，但它只是LLMs领域未来创新路径中的一个小步骤。虽然我们无法预测未来，但有一些技术趋势将的爱我们走向创新之路。我们来看看它们具体是什么：

### 1. 自我改进的自主模型

这些LLMs可能会有能力生成自己的训练数据以提高自己的表现。这在互联网上可用的大量信息被耗尽后可能特别有帮助。作为一个最近的例子，Google的LLM能够生成自己的问题和答案，然后相应地进行自我微调。

### 2. 能够验证自己输出的模型

能够为它们生成的信息提供来源的LLMs可以为整个技术增加更大的可信度。例如，OpenAI的WebGPT能够生成准确、详细的响应，并有来源作为支持。

### 3. 稀疏专家模型的发展

今天的最知名的LLMs有几个共同特点：它们是基于变换器架构的密集、自监督、预训练模型。然而，稀疏专家模型正在将技术引向另一个方向。有了这些模型，只需要激活相关的参数，使它们更大、更复杂。同时，它们需要更少的资源和能源消耗来进行模型训练。

### 关键要点

总之，LLMs是基础模型，它是一种大型神经网络，可以生成或嵌入文本。随着LLMs的扩展，它们可以解锁新的能力，例如翻译外语、编写代码等。它们只需在模型训练期间观察语言中的重复模式。这项技术确实令人惊叹！但随着这项创新，也带来了一些问题。例如，考虑到人类感知的主观性，RLHF在模型训练期间可能会无意中将偏见带入像GPT这样的AI产品中。

以下是一些关键总结：

- 拥有从20世纪50年代和60年代开始的历史，LLMs直到最近才随着ChatGPT的引入而成为家喻户晓的名字。
- LLMs有许多应用，包括情感分析、全文生成、分类、文本编辑、语言翻译、信息提取和摘要。
- 更大并不总是意味着更好：研究人员发现，最终更小、更优化训练的模型在性能上超过了它们的巨型对手，并且需要更少的能源和资源。
- 尽管LLMs有能力帮助我们解决许多现实世界的问题，但在使它们变得可靠、值得信赖和安全方面，我们还有很长的路要走。
- 基于LLM的GPT产品可以像人类一样学习和交流，但我们的学习和行为模式仍然要复杂得多。因此，尽管各行各业的专业人士可能对工作安全感到担忧，学术界对论文变得过时感到紧张，但在不确定性中，人们对无限潜力和无数机会感到兴奋不已。

要了解更多关于LLMs的信息，请查看我们的博客，我们将涵盖AI、机器学习和这项创新技术不断发展的所有细节。





翻译自：[The history, timeline, and future of LLMs](https://toloka.ai/blog/history-of-llms/)

