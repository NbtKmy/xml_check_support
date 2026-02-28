# XML Check Support — 開発メモ

## アプリ概要

OAI-PMH形式のXMLファイルをブラウザ内でチェック・閲覧するサポートアプリ。
Vanilla JS のシングルHTML（`index.html`）で完結。外部サーバへのデータ送信なし。

## XMLの構造（0000.xml を基準）

### フォーマット

**OAI-PMH 2.0** + **Dublin Core (dcndl)** + **RDF**

国立国会図書館サーチ（NDLサーチ）の OAI-PMH API レスポンス。

### ルート構造

```xml
<OAI-PMH xmlns="http://www.openarchives.org/OAI/2.0/" ...>
  <responseDate>2025-02-20T17:29:04Z</responseDate>
  <request verb="ListRecords" metadataPrefix="dcndl" ...>https://ndlsearch.ndl.go.jp/api/oaipmh</request>
  <ListRecords>
    <record> ... </record>
    <record> ... </record>
    ...
  </ListRecords>
</OAI-PMH>
```

### 1レコードの構造

```xml
<record>
  <header>
    <identifier>oai:ndlsearch.ndl.go.jp:R100000145-I000006044</identifier>
    <datestamp>2023-12-16T09:29:59Z</datestamp>
    <setSpec>book</setSpec>
    <setSpec>minamata</setSpec>
  </header>
  <metadata>
    <rdf:RDF xmlns:rdfs="..." xmlns:dc="..." xmlns:dcterms="..." xmlns:dcndl="..."
             xmlns:foaf="..." xmlns:owl="..." xmlns:rdf="...">

      <!-- 管理リソース -->
      <dcndl:BibAdminResource rdf:about="https://ndlsearch.ndl.go.jp/books/R100000145-I000006044">
        <dcterms:description>type : book</dcterms:description>
        <dcndl:bibRecordCategory>R100000145</dcndl:bibRecordCategory>
        <dcndl:record rdf:resource="...#material"/>
      </dcndl:BibAdminResource>

      <!-- 書誌リソース（メイン） -->
      <dcndl:BibResource rdf:about="...#material">
        <dcterms:title>水俣工場電解改良電極ピン之図</dcterms:title>
        <dc:title>
          <rdf:Description><rdf:value>水俣工場電解改良電極ピン之図</rdf:value></rdf:Description>
        </dc:title>
        <dcterms:creator>
          <foaf:Agent><foaf:name>下田一夫</foaf:name></foaf:Agent>
        </dcterms:creator>
        <dc:creator>下田一夫</dc:creator>
        <dcterms:date>1963. 1. 5</dcterms:date>
        <dcterms:issued rdf:datatype="http://purl.org/dc/terms/W3CDTF">1963-01-05</dcterms:issued>
        <dcterms:description>拠点名: 正門</dcterms:description>
        <dcterms:description>写真種別: ライカ</dcterms:description>
        <dcterms:description>画像有無: ○</dcterms:description>
        <dcterms:subject>
          <rdf:Description><rdf:value>40 第二組合就労行動</rdf:value></rdf:Description>
        </dcterms:subject>
        <dcterms:language rdf:datatype="http://purl.org/dc/terms/ISO639-2">jpn</dcterms:language>
        <dcterms:extent>46×39</dcterms:extent>
        <dcndl:materialType rdf:resource="http://ndl.go.jp/ndltype/Photograph" rdfs:label="写真"/>
        <dcndl:materialType rdf:resource="http://purl.org/dc/dcmitype/StillImage" rdfs:label="静止画資料"/>
        <dcterms:spatial>チッソ水俣工場正門</dcterms:spatial>
        <dcterms:temporal>1963. 1. 5</dcterms:temporal>
        <dcterms:audience>一般</dcterms:audience>
        <rdfs:seeAlso rdf:resource="https://gkbn.kumagaku.ac.jp/minamata/db/..."/>
      </dcndl:BibResource>

      <!-- 書誌リソース（アイテムへの参照） -->
      <dcndl:BibResource rdf:about="...#material">
        <dcndl:record rdf:resource="...#item"/>
      </dcndl:BibResource>

      <!-- アイテムリソース -->
      <dcndl:Item rdf:about="...#item">
        <rdfs:seeAlso rdf:resource="https://gkbn.kumagaku.ac.jp/minamata/db/..."/>
      </dcndl:Item>

    </rdf:RDF>
  </metadata>
</record>
```

### フィールド一覧

| フィールド | XPath（レコード内） | 備考 |
|---|---|---|
| タイトル | `.//dcterms:title` | `dc:title/rdf:Description/rdf:value` にも重複あり |
| 作成者 | `.//foaf:name`, `.//dc:creator` | 両方に同じ値が入ることが多い |
| 日付（表示用） | `.//dcterms:date` | 「1963. 1. 5」など自由記述 |
| 日付（機械可読） | `.//dcterms:issued` | W3CDTF形式（例: `1963-01-05`） |
| 説明 | `.//dcterms:description` | 複数あり。「type : book」「拠点名: ○○」など |
| 主題 | `.//dcterms:subject//rdf:value` | |
| 言語 | `.//dcterms:language` | ISO639-2（例: `jpn`） |
| 数量・大きさ | `.//dcterms:extent` | 複数あり |
| 場所 | `.//dcterms:spatial` | |
| 時間的範囲 | `.//dcterms:temporal` | |
| 対象読者 | `.//dcterms:audience` | ほぼ「一般」 |
| 資料種別 | `.//dcndl:materialType[@rdfs:label]` | 複数あり。属性値を使う |
| 外部リンク | `.//rdfs:seeAlso[@rdf:resource]` | 複数あり。属性値を使う |
| 識別子 | `header > identifier` | `oai:ndlsearch.ndl.go.jp:R100000145-XXXXXXXXX` |
| 更新日時 | `header > datestamp` | ISO 8601 |
| セット | `header > setSpec` | 複数あり（例: `book`, `minamata`） |
| 書誌カテゴリ | `.//dcndl:bibRecordCategory` | `R100000145` など |

### 注意点

- 名前空間が多いため、`querySelector` では取れない要素がある → `xml.evaluate`（XPath）を使う
- `dcterms:creator` は `foaf:Agent/foaf:name` のラッパーで同じ値が重複するため `Set` で重複除去する
- `dcterms:description` の「type : book」は `BibAdminResource` 側に入っており、書誌の説明とは別
- `rdfs:seeAlso` は `BibResource` と `Item` の両方に同じURLが入ることが多い → `Set` で重複除去する
- ファイルサイズが大きい（0000.xml は約 661KB）

## 技術スタック

- Vanilla JS（フレームワークなし）
- ブラウザ標準の `DOMParser` / `XPathResult` でXML解析
- シングルHTMLファイル（`index.html`）
