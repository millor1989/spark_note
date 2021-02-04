### 特征提取算法（Feature Extractors）

#### 1、TF-IDF

词频逆文本频率（TF-IDF）是一个特征向量化方法，广泛地运用在反映语料库中一个词对一篇文档的重要性的文本挖掘中。用`t`表示一个词，`d`表示一个文档，`D`表示语料库。词频`TF(t, d)`是文档`d`中词`t`出现的次数，而文档频率`DF(t, D)`表示包含词`t`的文档的数量。如果只使用词频来衡量重要性，很容易就会过于强调那些经常出现但是携带很少文档信息的词，比如，“a”，“the”，“of”等。如果整个语料库中一个词经常出现，这意味着它不携带关于某个特定文档特别信息。逆文档频率`IDF(t, D)`是一个词携带多少信息的数值量度，一个词在所有文档中出现的次数越多越小，如果出现在所有的文档中则`IDF`为0。TF-IDF是`TF`和`IDF`的乘积。有几个基于词频和逆文档频率的定义算法变种。在MLlib中，为了灵活将**TF**和**IDF**分开了。

**TF**：`HashingTF`和`CountVectorizer`都可以用来生成词频向量。

`HashingTF`是一个`Transformer`，可以将词集合转换为定长的特征向量。在文本处理中，“词集合”可能是一包单词。`HashingTF`使用了哈希技术。通过使用哈希函数，原始特征被映射为一个索引（词）。这里使用的哈希函数是MurmurHash 3。然后，基于映射的索引计算词频。这种方法避免了进行全局的词到索引的映射（对于大型语料库开销比较大），但是有潜在的哈希冲撞风险（不同的原始特征在哈希后可能变成相同的词）。为了减小哈希冲撞概率，可以增加目标特征的维度，即，哈希表的桶的数量。因为一个简单的模是用于将哈希函数转换为列索引的，建议使用2的指数作为特征维度，否则特征不会被均匀地映射到列默认的特征维度是2^18。一个可选的二进制开关参数（binary toggle parameter）控制了词频的计数（counts），当设置为true时所有的非0频率计数都被设置为1，这对于那些对二进制计数而不是整数计数的离散概率模型尤其有用。

`CountVectorizer`将文本文档转换为词数量的向量。

**IDF**：`IDF`是一个`Estimator`，基于`Dataset`拟合出一个`IDFModel`。`IDFModel`输入特征向量（通常是`HashingTF`或`CountVectorizer`创建的）并缩放对每列（scales each column）。直观上说，它能降低语料库中频繁出现的列的权重。

##### 例

使用`Tokenizer`将每个语句切分为单词。对于每个语句（一组单词），用`HashingTF`将语句哈希化为特征向量。使用`IDF`重新缩放特征向量；使用文本作为特征时，这样通常可以提升表现。然后将特征向量传递给学习算法。

```scala
import org.apache.spark.ml.feature.{HashingTF, IDF, Tokenizer}

val sentenceData = spark.createDataFrame(Seq(
  (0.0, "Hi I heard about Spark"),
  (0.0, "I wish Java could use case classes"),
  (1.0, "Logistic regression models are neat")
)).toDF("label", "sentence")

val tokenizer = new Tokenizer().setInputCol("sentence").setOutputCol("words")
val wordsData = tokenizer.transform(sentenceData)

val hashingTF = new HashingTF()
  .setInputCol("words").setOutputCol("rawFeatures").setNumFeatures(20)

val featurizedData = hashingTF.transform(wordsData)
// alternatively, CountVectorizer can also be used to get term frequency vectors

val idf = new IDF().setInputCol("rawFeatures").setOutputCol("features")
val idfModel = idf.fit(featurizedData)

val rescaledData = idfModel.transform(featurizedData)
rescaledData.select("label", "features").show()
```

#### 2、Word2Vec

`Word2Vec`是一个`Estimator`，输入代表文档的一系列单词，训练一个`Word2VecModel`。这个模型将每个单词映射为一个唯一地固定大小的（fixed-sized）向量。`Word2VecModel`文档中的所有单词的平均值将文档转换为一个向量；这个向量可以用作特征进行预测，文档相似度计算，等等。

`Word2Vec`计算表示单词的分布式向量（distributed vector）。分布式向量表示的优势是相似的单词在向量空间中是接近的，这会使新模式（novel pattern）的泛化（generalization，一般化）更简单，并且使模型估计（model estimation）更加健壮。在许多的自然语言处理应用（named entity recognition，消除歧义：disambiguation，parsing，tagging and machine translation）中，分布式向量表示都是有用的。

在MLlib的Word2Vec实现中，使用的是skip-gram模型。skip-gram的训练目标是学习擅长预测相同语句中单词上下文的单词向量表示。在skip-gram模型中，每个单词有两个相关的向量，分别是单词和上下文的向量表示。通过给定的单词预测到正确单词的概率由softmax模型决定。使用softmax模型的skip-gram模型是高开销的，为了加快Word2Vec的训练速度，MLlib使用的是hierarchical softmax，降低了运算的复杂度。

##### 例

有一组的文档，每个文档都是一个单词序列。将每个文档转换为一个可以传递给学习算法的特征向量。

```scala
import org.apache.spark.ml.feature.Word2Vec
import org.apache.spark.ml.linalg.Vector
import org.apache.spark.sql.Row

// Input data: Each row is a bag of words from a sentence or document.
val documentDF = spark.createDataFrame(Seq(
  "Hi I heard about Spark".split(" "),
  "I wish Java could use case classes".split(" "),
  "Logistic regression models are neat".split(" ")
).map(Tuple1.apply)).toDF("text")

// Learn a mapping from words to Vectors.
val word2Vec = new Word2Vec()
  .setInputCol("text")
  .setOutputCol("result")
  .setVectorSize(3)
  .setMinCount(0)
val model = word2Vec.fit(documentDF)

val result = model.transform(documentDF)
result.collect().foreach { case Row(text: Seq[_], features: Vector) =>
  println(s"Text: [${text.mkString(", ")}] => \nVector: $features\n") }
```

#### 3、CountVectorizer

`CountVectorizer`和`CountVectorizerModel`的目标是将一组文本文档转换为token数量的向量。当先验的（a-priori）字典不可用时，`CountVectorizer`可以用作`Estimator`来提取词汇表，并生成一个`CountVectorizer`。这个模型产生基于词汇表的文档稀疏表示，这种表示可以传递给像LDA一样的其它算法。

在拟合过程中，`CountVectorizer`会按照整个语料库中单词的频率筛选前`vocabSize`个单词。可选参数`minDF`指定包含词汇表中某一个单词的文档的最小数量。还有一个控制输出向量的二进制开关参数，如果设置为true，所有的非零数量都被设置为1，这对于那些对二进制计数而不是整数计数的离散概率模型尤其有用。

##### 例

假如有一个列为`id`和`texts`的DataFrame：

```
 id | texts
----|----------
 0  | Array("a", "b", "c")
 1  | Array("a", "b", "b", "c", "a")
```

对它调用`CountVectorizer`的fit方法会生成一个词汇表为（a，b，c）`CountVectorizerModel`。那么，包含转换后的输出为：

```
 id | texts                           | vector
----|---------------------------------|---------------
 0  | Array("a", "b", "c")            | (3,[0,1,2],[1.0,1.0,1.0])
 1  | Array("a", "b", "b", "c", "a")  | (3,[0,1,2],[2.0,2.0,1.0])
```

每个向量表示词汇表中在文档中出现的单词的数量。

```scala
import org.apache.spark.ml.feature.{CountVectorizer, CountVectorizerModel}

val df = spark.createDataFrame(Seq(
  (0, Array("a", "b", "c")),
  (1, Array("a", "b", "b", "c", "a"))
)).toDF("id", "words")

// fit a CountVectorizerModel from the corpus
val cvModel: CountVectorizerModel = new CountVectorizer()
  .setInputCol("words")
  .setOutputCol("features")
  .setVocabSize(3)
  .setMinDF(2)
  .fit(df)

// alternatively, define CountVectorizerModel with a-priori vocabulary
val cvm = new CountVectorizerModel(Array("a", "b", "c"))
  .setInputCol("words")
  .setOutputCol("features")

cvModel.transform(df).show(false)
```

