---
layout:     post
title:      "【Java】基于Lucene 7.1.0实现搜索引擎"
tags:
    - Java
    - Lucene
---

## 引入maven依赖
```
<!-- lucene -->
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-core</artifactId>
    <version>7.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-analyzers-smartcn</artifactId>
    <version>7.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-highlighter</artifactId>
    <version>7.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-backward-codecs</artifactId>
    <version>7.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-suggest</artifactId>
    <version>7.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-queryparser</artifactId>
    <version>7.1.0</version>
</dependency>

<!-- ikanalyzer -->
<dependency>
    <groupId>com.janeluo</groupId>
    <artifactId>ikanalyzer</artifactId>
    <version>2012_u6</version>
</dependency>
```

## IK分词组件
```
public class IKTokenizer extends Tokenizer {

    /**
     * IK分词器实现
     */
    private IKSegmenter ikimplement;

    /**
     * 词元文本属性
     */
    private final CharTermAttribute termAtt;
    /**
     * 词元位移属性
     */
    private final OffsetAttribute offsetAtt;
    /**
     * 词元分类属性（该属性分类参考org.wltea.analyzer.core.Lexeme中的分类常量）
     */
    private final TypeAttribute typeAtt;
    /**
     * 记录最后一个词元的结束位置
     */
    private int endPosition;

    IKTokenizer(Reader in) {
        this(in, false);
    }

    private IKTokenizer(Reader in, boolean useSmart) {
        offsetAtt = addAttribute(OffsetAttribute.class);
        termAtt = addAttribute(CharTermAttribute.class);
        typeAtt = addAttribute(TypeAttribute.class);
        ikimplement = new IKSegmenter(in, useSmart);
    }

    @Override
    public boolean incrementToken() throws IOException {
        // 清除所有的词元属性
        clearAttributes();
        Lexeme nextLexeme = ikimplement.next();
        if (nextLexeme != null) {
            // 将Lexeme转成Attributes
            // 设置词元文本
            termAtt.append(nextLexeme.getLexemeText());
            // 设置词元长度
            termAtt.setLength(nextLexeme.getLength());
            // 设置词元位移
            offsetAtt.setOffset(nextLexeme.getBeginPosition(),
                    nextLexeme.getEndPosition());
            // 记录分词的最后位置
            endPosition = nextLexeme.getEndPosition();
            // 记录词元分类
            typeAtt.setType(nextLexeme.getLexemeTypeString());
            // 返会true告知还有下个词元
            return true;
        }
        // 返会false告知词元输出完毕
        return false;
    }

    @Override
    public void reset() throws IOException {
        super.reset();
        ikimplement.reset(input);
    }

    @Override
    public final void end() {
        // set final offset
        int finalOffset = correctOffset(this.endPosition);
        offsetAtt.setOffset(finalOffset, finalOffset);
    }
}
```

## IK分词
```
public class IKAnalyzer extends StopwordAnalyzerBase {

    @Override
    protected TokenStreamComponents createComponents(String arg0) {
        try {
            IKTokenizer tokenizer = new IKTokenizer(new StringReader(arg0));
            TokenStream stream = new StandardFilter(tokenizer);
            stream = new LowerCaseFilter(stream);
            stream = new StopFilter(stream, getStopwordSet());
            return new TokenStreamComponents(tokenizer, stream);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

## 搜索纠错
```
public class CnSpellChecker implements Closeable {

    private Directory spellIndex;
    private float bStart;
    private float bEnd;
    private IndexSearcher searcher;
    private final Object searcherLock;
    private final Object modifyCurrentIndexLock;
    private volatile boolean closed;
    private float accuracy;
    private StringDistance sd;
    private Comparator<SuggestWord> comparator;

    private CnSpellChecker(Directory spellIndex, StringDistance sd) throws IOException {
        this(spellIndex, sd, SuggestWordQueue.DEFAULT_COMPARATOR);
    }

    CnSpellChecker(Directory spellIndex) throws IOException {
        this(spellIndex, new LevensteinDistance());
    }

    private CnSpellChecker(Directory spellIndex, StringDistance sd, Comparator<SuggestWord> comparator) throws IOException {
        this.bStart = 2.0F;
        this.bEnd = 1.0F;
        this.searcherLock = new Object();
        this.modifyCurrentIndexLock = new Object();
        this.closed = false;
        this.accuracy = 0.5F;
        this.setSpellIndex(spellIndex);
        this.setStringDistance(sd);
        this.comparator = comparator;
    }

    private void setSpellIndex(Directory spellIndexDir) throws IOException {
        synchronized (this.modifyCurrentIndexLock) {
            this.ensureOpen();
            if (!DirectoryReader.indexExists(spellIndexDir)) {
                IndexWriter writer = new IndexWriter(spellIndexDir, new IndexWriterConfig(null));
                writer.close();
            }

            this.swapSearcher(spellIndexDir);
        }
    }

    public void setComparator(Comparator<SuggestWord> comparator) {
        this.comparator = comparator;
    }

    public Comparator<SuggestWord> getComparator() {
        return this.comparator;
    }

    public void setStringDistance(StringDistance sd) {
        this.sd = sd;
    }

    public StringDistance getStringDistance() {
        return this.sd;
    }

    public void setAccuracy(float acc) {
        this.accuracy = acc;
    }

    public float getAccuracy() {
        return this.accuracy;
    }

    public String[] suggestSimilar(String word, int numSug) throws IOException {
        return this.suggestSimilar(word, numSug, (IndexReader) null, (String) null, SuggestMode.SUGGEST_WHEN_NOT_IN_INDEX);
    }

    public String[] suggestSimilar(String word, int numSug, float accuracy) throws IOException {
        return this.suggestSimilar(word, numSug, (IndexReader) null, (String) null, SuggestMode.SUGGEST_WHEN_NOT_IN_INDEX, accuracy);
    }

    public String[] suggestSimilar(String word, int numSug, IndexReader ir, String field, SuggestMode suggestMode) throws IOException {
        return this.suggestSimilar(word, numSug, ir, field, suggestMode, this.accuracy);
    }

    public String[] suggestSimilar(String word, int numSug, IndexReader ir, String field, SuggestMode suggestMode, float accuracy) throws IOException {
        IndexSearcher indexSearcher = this.obtainSearcher();

        try {
            if (ir == null || field == null) {
                suggestMode = SuggestMode.SUGGEST_ALWAYS;
            }

            if (suggestMode == SuggestMode.SUGGEST_ALWAYS) {
                ir = null;
                field = null;
            }

            int lengthWord = word.length();
            int freq = ir != null ? ir.docFreq(new Term(field, word)) : 0;
            int goalFreq = suggestMode == SuggestMode.SUGGEST_MORE_POPULAR ? freq : 0;
            if (suggestMode == SuggestMode.SUGGEST_WHEN_NOT_IN_INDEX && freq > 0) {
                return new String[]{word};
            } else {
                BooleanQuery.Builder query = new BooleanQuery.Builder();

                int ng;
                for (ng = getMin(lengthWord); ng <= getMax(lengthWord); ++ng) {
                    String key = "gram" + ng;
                    String[] grams = formGrams(word, ng);
                    if (grams.length != 0) {
                        if (this.bStart > 0.0F) {
                            add(query, "start" + ng, grams[0], this.bStart);
                        }

                        if (this.bEnd > 0.0F) {
                            add(query, "end" + ng, grams[grams.length - 1], this.bEnd);
                        }

                        for (String gram : grams) {
                            add(query, key, gram);
                        }
                    }
                }

                ng = 10 * numSug;
                ScoreDoc[] hits = indexSearcher.search(query.build(), ng).scoreDocs;
                SuggestWordQueue sugQueue = new SuggestWordQueue(numSug, this.comparator);
                int stop = Math.min(hits.length, ng);
                SuggestWord sugWord = new SuggestWord();

                for (int i = 0; i < stop; ++i) {
                    sugWord.string = indexSearcher.doc(hits[i].doc).get("word");
                    sugWord.score = this.sd.getDistance(word, sugWord.string);
                    if (sugWord.score >= accuracy) {
                        if (ir != null) {
                            sugWord.freq = ir.docFreq(new Term(field, sugWord.string));
                            boolean exits = suggestMode == SuggestMode.SUGGEST_MORE_POPULAR && goalFreq > sugWord.freq || sugWord.freq < 1;
                            if (exits) {
                                continue;
                            }
                        }

                        sugQueue.insertWithOverflow(sugWord);
                        if (sugQueue.size() == numSug) {
                            accuracy = sugQueue.top().score;
                        }

                        sugWord = new SuggestWord();
                    }
                }

                String[] list = new String[sugQueue.size()];

                for (int i = sugQueue.size() - 1; i >= 0; --i) {
                    list[i] = sugQueue.pop().string;
                }

                return list;
            }
        } finally {
            this.releaseSearcher(indexSearcher);
        }
    }

    private static void add(BooleanQuery.Builder q, String name, String value, float boost) {
        Query tq = new TermQuery(new Term(name, value));
        q.add(new BooleanClause(new BoostQuery(tq, boost), BooleanClause.Occur.SHOULD));
    }

    private static void add(BooleanQuery.Builder q, String name, String value) {
        q.add(new BooleanClause(new TermQuery(new Term(name, value)), BooleanClause.Occur.SHOULD));
    }

    private static String[] formGrams(String text, int ng) {
        int len = text.length();
        String[] res = new String[len - ng + 1];

        for (int i = 0; i < len - ng + 1; ++i) {
            res[i] = text.substring(i, i + ng);
        }

        return res;
    }

    public void clearIndex() throws IOException {
        synchronized (this.modifyCurrentIndexLock) {
            this.ensureOpen();
            Directory dir = this.spellIndex;
            IndexWriter writer = new IndexWriter(dir, (new IndexWriterConfig((Analyzer) null)).setOpenMode(IndexWriterConfig.OpenMode.CREATE));
            writer.close();
            this.swapSearcher(dir);
        }
    }

    public boolean exist(String word) throws IOException {
        IndexSearcher indexSearcher = this.obtainSearcher();

        boolean var3;
        try {
            var3 = indexSearcher.getIndexReader().docFreq(new Term("word", word)) > 0;
        } finally {
            this.releaseSearcher(indexSearcher);
        }

        return var3;
    }

    final void indexDictionary(Dictionary dict, IndexWriterConfig config) throws IOException {
        synchronized (this.modifyCurrentIndexLock) {
            this.ensureOpen();
            Directory dir = this.spellIndex;
            IndexWriter writer = new IndexWriter(dir, config);
            IndexSearcher indexSearcher = this.obtainSearcher();
            List<TermsEnum> termsEnums = new ArrayList<>();
            IndexReader reader = this.searcher.getIndexReader();
            if (reader.maxDoc() > 0) {
                for (LeafReaderContext ctx : reader.leaves()) {
                    Terms terms = ctx.reader().terms("word");
                    if (terms != null) {
                        termsEnums.add(terms.iterator());
                    }
                }
            }

            boolean isEmpty = termsEnums.isEmpty();

            try {
                InputIterator iterator = dict.getEntryIterator();
                label116:
                while (true) {
                    String word;
                    int len;

                    label114:
                    while (true) {
                        BytesRef currentTerm;
                        if ((currentTerm = iterator.next()) == null) {
                            break label116;
                        }
                        word = currentTerm.utf8ToString();
                        len = word.length();
                        if (word.matches("\\w+")) {
                            if (len < 3) {
                                continue;
                            }
                        } else {
                            if (len < 2) {
                                continue;
                            }
                        }
                        if (isEmpty) {
                            break;
                        }
                        Iterator var15 = termsEnums.iterator();

                        while (true) {
                            if (!var15.hasNext()) {
                                break label114;
                            }

                            TermsEnum te = (TermsEnum) var15.next();
                            if (te.seekExact(currentTerm)) {
                                break;
                            }
                        }
                    }
                    Document doc = createDocument(word, getMin(len), getMax(len));
                    writer.addDocument(doc);
                }
            } finally {
                this.releaseSearcher(indexSearcher);
            }

            if (false) {
                writer.forceMerge(1);
            }
            writer.close();
            this.swapSearcher(dir);
        }
    }

    private static int getMin(int l) {
        if (l > 5) {
            return 3;
        } else {
            return l == 5 ? 2 : 1;
        }
    }

    private static int getMax(int l) {
        if (l > 5) {
            return 4;
        } else {
            return l == 5 ? 3 : 2;
        }
    }

    private static Document createDocument(String text, int ng1, int ng2) {
        Document doc = new Document();
        Field f = new StringField("word", text, Field.Store.YES);
        doc.add(f);
        addGram(text, doc, ng1, ng2);
        return doc;
    }

    private static void addGram(String text, Document doc, int ng1, int ng2) {
        int len = text.length();

        for (int ng = ng1; ng <= ng2; ++ng) {
            String key = "gram" + ng;
            String end = null;

            for (int i = 0; i < len - ng + 1; ++i) {
                String gram = text.substring(i, i + ng);
                FieldType ft = new FieldType(StringField.TYPE_NOT_STORED);
                ft.setIndexOptions(IndexOptions.DOCS_AND_FREQS);
                Field ngramField = new Field(key, gram, ft);
                doc.add(ngramField);
                if (i == 0) {
                    Field startField = new StringField("start" + ng, gram, Field.Store.NO);
                    doc.add(startField);
                }
                end = gram;
            }

            if (end != null) {
                Field endField = new StringField("end" + ng, end, Field.Store.NO);
                doc.add(endField);
            }
        }

    }

    private IndexSearcher obtainSearcher() {
        synchronized (this.searcherLock) {
            this.ensureOpen();
            this.searcher.getIndexReader().incRef();
            return this.searcher;
        }
    }

    private void releaseSearcher(IndexSearcher aSearcher) throws IOException {
        aSearcher.getIndexReader().decRef();
    }

    private void ensureOpen() {
        if (this.closed) {
            throw new AlreadyClosedException("Spellchecker has been closed");
        }
    }

    @Override
    public void close() throws IOException {
        synchronized (this.searcherLock) {
            this.ensureOpen();
            this.closed = true;
            if (this.searcher != null) {
                this.searcher.getIndexReader().close();
            }

            this.searcher = null;
        }
    }

    private void swapSearcher(Directory dir) throws IOException {
        IndexSearcher indexSearcher = this.createSearcher(dir);
        synchronized (this.searcherLock) {
            if (this.closed) {
                indexSearcher.getIndexReader().close();
                throw new AlreadyClosedException("Spellchecker has been closed");
            } else {
                if (this.searcher != null) {
                    this.searcher.getIndexReader().close();
                }

                this.searcher = indexSearcher;
                this.spellIndex = dir;
            }
        }
    }

    private IndexSearcher createSearcher(Directory dir) throws IOException {
        return new IndexSearcher(DirectoryReader.open(dir));
    }

    boolean isClosed() {
        return this.closed;
    }
}
```

## Lucene
```
public class Lucene {

    /**
     * 创建索引
     *
     * @param path    索引存放位置
     * @param id      ID
     * @param title   标题
     * @param content 内容
     * @param time    时间
     */
    public void createIndex(Path path, Integer id, String title, String content, Long time) {
        // 使用的分词器
        IndexWriterConfig config = new IndexWriterConfig(new IKAnalyzer());
        // 索引存放磁盘位置
        try (FSDirectory directory = FSDirectory.open(path);
             IndexWriter writer = new IndexWriter(directory, config)) {
            // 建立文件索引
            Document document = new Document();
            // int类型 存放形式
            document.add(new IntPoint("id", id));
            // 保存存储信息，不写不保存存储信息
            document.add(new StoredField("id", id));

            // 字段名字，字段内容，Store:如果是yes 则说明存储到文档中
            document.add(new TextField("title", title, Field.Store.YES));
            document.add(new TextField("content", content, Field.Store.YES));

            document.add(new LongPoint("time", time));
            document.add(new StoredField("time", time));
            // 控制排序字段
            document.add(new NumericDocValuesField("time", time));

            writer.addDocument(document);
            // 必须存在，不然不生效
            writer.commit();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 修改索引
     *
     * @param path    索引存放位置
     * @param id      ID
     * @param content 内容
     */
    public void updateIndex(Path path, Integer id, String content) {
        // 使用的分词器
        IndexWriterConfig config = new IndexWriterConfig(new IKAnalyzer());
        // 索引存放磁盘位置
        try (FSDirectory directory = FSDirectory.open(path);
             IndexReader reader = DirectoryReader.open(directory);
             IndexWriter writer = new IndexWriter(directory, config)) {
            // 通过id查询对应的数据
            IndexSearcher search = new IndexSearcher(reader);
            TopDocs topDocs = search.search(IntPoint.newExactQuery("id", id), 1);
            if (topDocs.totalHits == 0) {
                return;
            }
            ScoreDoc scoreDoc = topDocs.scoreDocs[0];
            Document document = search.doc(scoreDoc.doc);

            document.add(new TextField("content", content, Field.Store.YES));

            // 通过id进行匹配，修改索引
            writer.updateDocument(new Term("id", String.valueOf(id)), document);
            writer.commit();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 删除索引
     *
     * @param path  索引存放位置
     * @param field 字段名称
     * @param value 关键词
     */
    public void deleteIndex(Path path, String field, String value) {
        IndexWriterConfig config = new IndexWriterConfig(new IKAnalyzer());
        // 索引存放磁盘位置
        try (FSDirectory directory = FSDirectory.open(path);
             IndexWriter writer = new IndexWriter(directory, config)) {
            // 通过匹配字段进行删除
            writer.deleteDocuments(new Term(field, value));
            writer.commit();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 同义词
     *
     * @param string 关键词
     * @return String
     */
    public static String synonym(String string) {
        List<String> list = new ArrayList<>();
        URL url = Lucene.class.getResource("/dic/synonym.dic");
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(new FileInputStream(url.getFile()), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                list.clear();
                list.addAll(Arrays.asList(line.split(" ")));
                if (list.contains(string)) {
                    reader.close();
                    list.remove(string);
                    return list.get(new Random().nextInt(list.size()));
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "";
    }

    /**
     * 搜索纠错
     *
     * @param path   索引存放位置
     * @param string 关键词
     * @return String
     */
    public static String correct(Path path, String string) {
        try (FSDirectory directory = FSDirectory.open(path)) {
            URL url = Lucene.class.getResource("/dic/correct.dic");
            if (url != null) {
                CnSpellChecker checker = new CnSpellChecker(directory);
                IndexWriterConfig config = new IndexWriterConfig(new IKAnalyzer());
                checker.indexDictionary(new PlainTextDictionary(new FileInputStream(url.getFile())), config);
                String[] strings = checker.suggestSimilar(string, 10, 0.5f);
                if (strings.length > 0) {
                    return strings[0];
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "";
    }

    /**
     * 拼音转文字
     *
     * @param string 关键词
     * @return String
     */
    public static String pinyin(String string) {
        URL url = Lucene.class.getResource("/dic/pinyin.dic");
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(new FileInputStream(url.getFile()), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.endsWith("-" + string)) {
                    reader.close();
                    return line.split("-")[0];
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "";
    }

    /**
     * 搜索
     *
     * @param path      索引存放位置
     * @param string    关键词
     * @param startTime 开始时间
     * @param endTime   结束时间
     * @param order     排序 0默认打分排序，1时间排序
     * @param reverse   升降序 0升序，1降序
     * @param page      页数
     * @param size      一页的大小
     * @return JSONObject
     */
    public JSONObject search(Path path, String string, Long startTime, Long endTime, Integer order, Boolean reverse, Integer page, Integer size) {
        List<String> list = new ArrayList<>();
        JSONObject object = new JSONObject();
        IKAnalyzer analyzer = new IKAnalyzer();
        // 索引存放磁盘位置
        try (FSDirectory directory = FSDirectory.open(path);
             IndexReader reader = DirectoryReader.open(directory)) {
            IndexSearcher search = new IndexSearcher(reader);
            BooleanQuery.Builder builder = new BooleanQuery.Builder();
            if (string == null || string.length() == 0) {
                string = "*";
                builder.add(new WildcardQuery(new Term("title", string)), BooleanClause.Occur.MUST);
            } else {
                BooleanQuery.Builder should = new BooleanQuery.Builder();
                // 设置匹配词的权重
                should.add(new BoostQuery(new QueryParser("title", analyzer).parse(string), 10f), BooleanClause.Occur.SHOULD);
                should.add(new BoostQuery(new QueryParser("content", analyzer).parse(string), 1f), BooleanClause.Occur.SHOULD);
                builder.add(should.build(), BooleanClause.Occur.MUST);
                list.add("content");
            }

            if (startTime != 0 && endTime != 0) {
                builder.add(LongPoint.newRangeQuery("time", startTime, endTime), BooleanClause.Occur.MUST);
            }
            BooleanQuery query = builder.build();

            Sort sort = null;
            SortField sortField;
            switch (order) {
                case 0:
                    sortField = new SortField(null, SortField.Type.SCORE, reverse);
                    sort = new Sort(sortField);
                    break;
                case 1:
                    sortField = new SortField("time", SortField.Type.LONG, reverse);
                    sort = new Sort(sortField);
                    break;
                default:
                    break;
            }
            // 通过indexSearcher来搜索索引
            TopDocs topDocs = search.search(query, page * size, sort, true, false);
            // 关键字高亮显示的html标签
            SimpleHTMLFormatter simpleHTMLFormatter = new SimpleHTMLFormatter("<span style='color:red'>", "</span>");
            Highlighter highlighter = new Highlighter(simpleHTMLFormatter, new QueryScorer(query));
            // 根据查询条件匹配出的记录总数
            long count = topDocs.totalHits;
            object.put("count", count);
            JSONArray array = new JSONArray();
            // 打分
            ScoreDoc[] scoreDocs = topDocs.scoreDocs;
            int start = page - 1;
            if (page > 1) {
                start = (page - 1) * size;
            }
            int end = page * size;
            if (end > count) {
                end = (int) count;
            }
            if (end > start) {
                for (int i = start; i < end; i++) {
                    JSONObject json = new JSONObject();
                    Document doc = search.doc(scoreDocs[i].doc);
                    for (IndexableField field : doc.getFields()) {
                        String name = field.name();
                        String value = field.stringValue();
                        if (list.contains(name)) {
                            // 内容增加高亮显示
                            TokenStream stream = analyzer.tokenStream(name, new StringReader(value));
                            String highlight = highlighter.getBestFragment(stream, value);
                            if (highlight == null) {
                                json.put(name, value);
                            } else {
                                json.put(name, highlight);
                            }
                        } else {
                            if (value.length() > 240) {
                                json.put(name, value.substring(0, 240));
                            } else {
                                json.put(name, value);
                            }
                        }
                    }
                    json.put("score", scoreDocs[i].score);
                    array.add(json);
                }
            }
            object.put("data", array);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return object;
    }

    /**
     * 相关文章推荐
     *
     * @param path 索引存放位置
     * @param id   ID
     * @param size 显示条数
     * @return JSONArray
     */
    public JSONArray moreLikeThis(Path path, String id, Integer size) {
        JSONArray array = new JSONArray();
        // 索引存放磁盘位置
        try (FSDirectory directory = FSDirectory.open(path);
             IndexReader reader = DirectoryReader.open(directory)) {
            IndexSearcher searcher = new IndexSearcher(reader);

            TopDocs doc = searcher.search(new TermQuery(new Term("file_id", id)), 1);
            if (doc.totalHits == 0) {
                return array;
            }
            MoreLikeThis mlt = new MoreLikeThis(reader);
            mlt.setMinTermFreq(1);
            mlt.setMinDocFreq(1);
            mlt.setAnalyzer(new IKAnalyzer());
            mlt.setFieldNames(new String[]{"title", "content"});
            Query query = mlt.like(doc.scoreDocs[0].doc);
            BooleanQuery.Builder builder = new BooleanQuery.Builder();
            builder.add(query, BooleanClause.Occur.MUST);
            builder.add(new BoostQuery(new WildcardQuery(new Term("title", "*")), 10f), BooleanClause.Occur.MUST);
            builder.add(new BoostQuery(new WildcardQuery(new Term("content", "*")), 1f), BooleanClause.Occur.MUST);
            BooleanQuery.Builder file = new BooleanQuery.Builder();
            TopDocs topDocs = searcher.search(builder.build(), size + 1, Sort.RELEVANCE, true, false);

            ScoreDoc[] scoreDocs = topDocs.scoreDocs;
            for (ScoreDoc scoreDoc : scoreDocs) {
                Document document = reader.document(scoreDoc.doc);
                String field = document.getField("id").stringValue();
                if (!field.equals(id) && scoreDoc.score > 60f) {
                    JSONObject json = new JSONObject();
                    json.put("id", field);
                    json.put("title", document.getField("title").stringValue());
                    json.put("content", document.getField("content").stringValue());
                    json.put("score", scoreDoc.score);
                    array.add(json);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return array;
    }
}
```
